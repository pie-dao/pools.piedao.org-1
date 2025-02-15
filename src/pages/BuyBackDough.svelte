<script>
  import { onMount } from 'svelte';
  import { eth, approveMax, subject } from '../stores/eth.js';
  import { getTokenImage } from '../components/helpers';
  import { fetchBalances } from '../helpers/multicall';
  import debounce from 'lodash/debounce';
  import { _ } from 'svelte-i18n';
  import BigNumber from 'bignumber.js';
  import displayNotification from '../notifications';
  import { ethers } from 'ethers';
  import buyBackAbi from '../abis/buyBackABI.json';
  import erc20Abi from '../abis/erc20ABI.json';
  import DoughABI from '../abis/DoughABI.json';
  import contracts from '../config/smartcontracts.json';
  import { formatFiat } from '../components/helpers.js';

  const baseListed = [
    {
      address: '0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee',
      symbol: 'ETH',
      icon: getTokenImage('eth'),
      decimals: 18,
    },
    {
      address: '0xad32A8e6220741182940c5aBF610bDE99E737b2D',
      symbol: 'DOUGH',
      icon: getTokenImage('0xad32A8e6220741182940c5aBF610bDE99E737b2D'),
      decimals: 18,
    },
    {
      address: '0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48',
      symbol: 'USDC',
      icon: getTokenImage('0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48'),
      decimals: 6,
    },
  ];

  $: listed = baseListed;

  let defaultTokenSell = baseListed.find((l) => l.symbol === 'DOUGH');
  let defaultTokenBuy = baseListed.find((l) => l.symbol === 'USDC');

  const defaultAmount = {
    bn: new BigNumber(0),
    label: 0,
  };

  const toNum = (num, decimals = 18) =>
    BigNumber(num.toString())
      .dividedBy(10 ** decimals)
      .toFixed(2);

  const toNumFixed = (num, decimals = 18) =>
    Number(
      BigNumber(num.toString())
        .dividedBy(10 ** decimals)
        .toFixed(4),
    );

  $: sellToken = defaultTokenSell;
  $: buyToken = defaultTokenBuy;
  $: amount = defaultAmount;
  $: burnAmount = defaultAmount;
  $: receivedAmount = 0;
  $: needAllowance = false;
  $: initialized = {
    onMount: false,
    onChainData: false,
  };
  $: isLoading = false;
  $: allowances = {};
  $: balances = {};
  $: error = null;
  $: balanceError = false;
  $: avaliableToBuy = 0;
  $: tokenPrice = 0;
  $: poolUSDC = 0;

  let buyBackContract;

  $: if ($eth.address) {
    if (!initialized.onChainData && !isLoading) {
      isLoading = true;
      fetchOnchainData();
      initialized.onChainData = true;
      isLoading = false;
    }
  }

  function needApproval(allowance) {
    if (!$eth.address || !$eth.signer) return false;
    if (allowance.isEqualTo(0)) return true;
    if (allowance.isGreaterThanOrEqualTo(amount.bn)) return false;
  }

  async function approveToken() {
    if (!$eth.address || !$eth.signer) {
      displayNotification({ message: $_('piedao.please.connect.wallet'), type: 'hint' });
      connectWeb3();
      return;
    }

    await approveMax(sellToken.address, contracts.buyBackDough);
    needAllowance = false;
    await fetchOnchainData();
  }

  async function onAmountChange() {
    const decimals = sellToken.decimals || 18;
    amount.bn = new BigNumber(amount.label).multipliedBy(10 ** decimals);
    receivedAmount = await estimateBuyback(amount.bn);
    needAllowance = needApproval(allowances[sellToken.address]);
  }

  async function burningAmountChange() {
    const decimals = sellToken.decimals || 18;
    burnAmount.bn = new BigNumber(burnAmount.label).multipliedBy(10 ** decimals);
  }

  onMount(async () => {
    isLoading = true;
    if ($eth.signer || $eth.provider) {
      buyBackContract = new ethers.Contract(
        contracts.buyBackDough,
        buyBackAbi,
        $eth.signer || $eth.provider,
      );
      await fetchBuyBack();
    }
    if ($eth.address) {
      await fetchOnchainData();
      initialized.onChainData = true;
    }

    initialized.onMount = true;
    isLoading = false;
  });

  async function fetchBuyBack() {
    const pricePerDough = await buyBackContract.price();
    try {
      const maxAvailableToBuy = await buyBackContract.maxAvailableToBuy();
      avaliableToBuy = toNum(maxAvailableToBuy, 18);
    } catch (e) {
      console.error(e);
    }
    const tokenOut = await buyBackContract.tokenOut();
    const tokenContract = new ethers.Contract(tokenOut, erc20Abi, $eth.signer || $eth.provider);
    const usdcBalance = await tokenContract.balanceOf(contracts.buyBackDough);
    poolUSDC = toNum(usdcBalance, 6);
    tokenPrice = toNumFixed(pricePerDough.value, 6);
  }

  async function fetchOnchainData() {
    // Fetch balances, allowance and decimals
    listed = await fetchBalances(listed, $eth.address, $eth.provider, contracts.buyBackDough);
    listed.forEach((token) => {
      allowances[token.address] = token.allowance;
    });

    listed.forEach((token) => {
      balances[token.address] = token.balance;
    });

    needAllowance = needApproval(allowances[sellToken.address]);
  }

  async function estimateBuyback(amountIn) {
    if (!amountIn.isGreaterThan(0)) return 0;
    const stringifyAmountBN = amountIn.toFixed(0);
    const { quoteTokenOut } = await buyBackContract.getBuybackQuote(stringifyAmountBN);
    return toNum(quoteTokenOut, 6);
  }

  async function swap() {
    if (!$eth.address || !$eth.signer) {
      displayNotification({ message: $_('piedao.please.connect.wallet'), type: 'hint' });
      connectWeb3();
      return;
    }

    const signedBuyBackContract = new ethers.Contract(
      contracts.buyBackDough,
      buyBackAbi,
      $eth.signer,
    );

    const { emitter } = displayNotification(
      await signedBuyBackContract.buyback(amount.bn.toFixed(0), $eth.address),
    );

    emitter.on('txConfirmed', ({ hash }) => {
      const { dismiss } = displayNotification({
        message: 'Confirming...',
        type: 'pending',
      });

      const subscription = subject('blockNumber').subscribe({
        next: async () => {
          displayNotification({
            autoDismiss: 15000,
            message: `${amount.label.toFixed(0)} ${sellToken.symbol} swapped successfully`,
            type: 'success',
          });
          await fetchOnchainData();
          amount = defaultAmount;
          dismiss();
          subscription.unsubscribe();
        },
      });

      return {
        autoDismiss: 1,
        message: 'Mined',
        type: 'success',
      };
    });
  }

  async function burn() {
    if (!$eth.address || !$eth.signer) {
      displayNotification({ message: $_('piedao.please.connect.wallet'), type: 'hint' });
      connectWeb3();
      return;
    }

    const signedDoughContract = new ethers.Contract(contracts.dough, DoughABI, $eth.signer);

    const { emitter } = displayNotification(
      await signedDoughContract.transfer(contracts.DEAD, burnAmount.bn.toFixed(0)),
    );

    emitter.on('txConfirmed', ({ hash }) => {
      const { dismiss } = displayNotification({
        message: 'Confirming...',
        type: 'pending',
      });

      const subscription = subject('blockNumber').subscribe({
        next: async () => {
          displayNotification({
            autoDismiss: 15000,
            message: `${burnAmount.label.toFixed(0)} ${sellToken.symbol} burned successfully`,
            type: 'success',
          });
          await fetchOnchainData();
          await fetchBuyBack();
          burnAmount = defaultAmount;
          dismiss();
          subscription.unsubscribe();
        },
      });

      return {
        autoDismiss: 1,
        message: 'Mined',
        type: 'success',
      };
    });
  }
</script>

<div class="relative content flex flex-col px-4">
  <div class="w-full py-20">
    <h1
      class="text-xl font-bold inline bg-gradient-to-r from-[#ed1ea0] via-[#6c5dfe] to-[#ed1ea0] bg-clip-text tracking-tight text-transparent leading-12"
    >
      DOUGH Buyback
    </h1>
    <div class="mt-6 space-y-6 font-display text-sm tracking-tight">
      <p>
        As per the PieDAO to Auxo migration, the Treasury Committee has determined a fixed price per
        epoch (2 weeks duration) in order for Dough holders to sell their tokens for stablecoins,
        being USDC the token of choice. If you want more information,
        <a
          class="text-pink"
          href="https://forum.piedao.org/t/dough-edough-buyback-program/1421"
          target="_blank"
          rel="noopener noreferrer">click here</a
        >. Below you will be able to check the pool's status and execute the swap if wanted.
      </p>
    </div>
    <div class="grid grid-cols-1 gap-6 sm:grid-cols-2 lg:grid-cols-3 mt-20">
      <div class="w-full rounded-xl flex cardbordergradient">
        <div class="px-6 py-4 flex flex-col">
          <div class="flex flex-col w-full">
            <span classy="text-base leading-6">Budget balance</span>
          </div>
          <div>
            <span class="font-thin text-lg mb-1 opacity-70"
              >{formatFiat(poolUSDC, ',', '.', '')} USDC</span
            >
          </div>
        </div>
      </div>
      <div class="w-full rounded-xl flex cardbordergradient">
        <div class="px-6 py-4 flex flex-col">
          <div class="flex flex-col w-full">
            <span classy="text-base leading-6">Current Epoch's DOUGH price</span>
          </div>
          <div>
            <span class="font-thin text-lg mb-1 opacity-70">{tokenPrice} USDC</span>
          </div>
        </div>
      </div>
      <div class="w-full rounded-xl flex cardbordergradient">
        <div class="px-6 py-4 flex flex-col">
          <div class="flex flex-col w-full">
            <span classy="text-base leading-6">Max you can sell</span>
          </div>
          <div>
            <span class="font-thin text-lg mb-1 opacity-70"
              >{formatFiat(avaliableToBuy, ',', '.', '')} DOUGH</span
            >
          </div>
        </div>
      </div>
    </div>
    <div
      class="cardboardergradient bg-transparent flex flex-col items-center w-94pc md:w-50pc h-50pc mx-auto mt-20"
    >
      <div class="w-full py-8 px-12 md:px-20 flex flex-col items-center">
        <h2 class="text-xl font-bold text-center mb-4">Sell DOUGH</h2>

        <div class="flex flex-col nowrap w-100pc swap-from border rounded-20px border-grey p-16px">
          <div class="flex items-center justify-between">
            <div class="flex nowrap intems-center p-1 font-thin">From</div>
            <div
              class="sc-kkGfuU hyvXgi css-1qqnh8x font-thin"
              style="display: inline; cursor: pointer;"
            >
              {#if balances[sellToken.address]}
                <div
                  on:click={async () => {
                    amount.label = balances[sellToken.address].number;
                    amount.bn = balances[sellToken.address].bn;
                    receivedAmount = await estimateBuyback(amount.bn);
                  }}
                >
                  Max balance: {balances[sellToken.address].label}
                </div>
              {/if}
            </div>
          </div>
          <div class="flex nowrap items-center p-1">
            <input
              class:error={balanceError}
              class="swap-input-from"
              on:focus={() => {
                amount.label = amount.label === 0 ? '' : amount.label;
              }}
              on:keyup={async () => {
                const debounced = debounce(onAmountChange, 400, {
                  leading: false,
                  trailing: true,
                });
                await debounced();
              }}
              bind:value={amount.label}
              inputmode="decimal"
              title="Token Amount"
              autocomplete="off"
              autocorrect="off"
              type="number"
              pattern="^[0-9]*[.]?[0-9]*$"
              placeholder="0.0"
              minlength="1"
              maxlength="79"
              spellcheck="false"
            />
            <button class="swap-button">
              <span class="sc-iybRtq gjVeBU">
                <img
                  class="h-auto w-24px mr-5px"
                  alt={sellToken ? `${sellToken.symbol} logo` : ''}
                  src={sellToken ? sellToken.icon : ''}
                />
                <span class="sc-kXeGPI jeVIZw token-symbol-container"
                  >{sellToken ? sellToken.symbol : ''}</span
                >
              </span>
            </button>
          </div>
        </div>

        <div class="my-20px flex items-center">
          <svg
            xmlns="http://www.w3.org/2000/svg"
            width="24"
            height="24"
            viewBox="0 0 24 24"
            fill="none"
            stroke="#cccccc"
            stroke-width="2"
            stroke-linecap="round"
            stroke-linejoin="round"
            ><line x1="12" y1="5" x2="12" y2="19" /><polyline points="19 12 12 19 5 12" /></svg
          >
        </div>

        <div class="flex flex-col nowrap w-100pc swap-from border rounded-20px border-grey p-16px">
          <div class="flex items-center justify-between">
            <div class="flex nowrap intems-center p-1 font-thin">To (estimated)</div>
          </div>
          <div class="flex nowrap items-center p-1">
            <input
              class="swap-input-from"
              bind:value={receivedAmount}
              disabled
              inputmode="decimal"
              title="Token Amount"
              autocomplete="off"
              autocorrect="off"
              type="text"
              pattern="^[0-9]*[.,]?[0-9]*$"
              placeholder="0.0"
              minlength="1"
              maxlength="79"
              spellcheck="false"
            />
            <button class="swap-button">
              <span class="sc-iybRtq gjVeBU">
                <img
                  class="h-auto w-24px mr-5px"
                  alt={buyToken ? `${buyToken.symbol} logo` : ''}
                  src={buyToken ? buyToken.icon : ''}
                />
                <span class="sc-kXeGPI jeVIZw token-symbol-container"
                  >{buyToken ? buyToken.symbol : ''}</span
                >
              </span>
            </button>
          </div>
        </div>
        {#if error}
          <button disabled={true} class="stake-button error rounded-20px mt-10px p-15px w-100pc">
            {error}
          </button>
        {:else if needAllowance}
          <button
            on:click={approveToken}
            class="btn clear stake-button mt-10px rounded-20px p-15px w-100pc">Approve DOUGH</button
          >
        {:else}
          <button
            class:error={error || isLoading}
            on:click={swap}
            disabled={error ||
              isLoading ||
              amount.label === 0 ||
              amount.label === '' ||
              !$eth.address}
            class="stake-button mt-10px rounded-20px p-15px w-100pc"
          >
            Sell
          </button>
        {/if}
      </div>
    </div>
    <div class="text-center my-4">
      <h2
        class="text-xl font-bold inline bg-gradient-to-r from-[#ed1ea0] via-[#6c5dfe] to-[#ed1ea0] bg-clip-text tracking-tight text-transparent leading-12 my-4"
      >
        OR
      </h2>
    </div>

    <div class="rainbow bg-transparent flex flex-col items-center w-94pc md:w-50pc h-50pc mx-auto">
      <h2 class="text-md font-medium text-left mb-4">
        Here you can burn your DOUGH, what do you get out of it? Our most sincere gratitude to
        facilitate the process. Good Karma to you anon. The universe does not carry debts. It always
        returns back to you what you gave it.
      </h2>
      <div class="flex flex-col nowrap w-100pc swap-from border rounded-20px border-grey p-16px">
        <div class="flex items-end">
          <div class="font-thin ml-auto cursor-pointer">
            {#if balances[sellToken.address]}
              <div
                on:click={async () => {
                  burnAmount.label = balances[sellToken.address].number;
                  burnAmount.bn = balances[sellToken.address].bn;
                }}
              >
                Max balance: {balances[sellToken.address].label}
              </div>
            {/if}
          </div>
        </div>
        <div class="flex nowrap items-center p-1">
          <input
            class:error={balanceError}
            class="swap-input-from"
            on:focus={() => {
              burnAmount.label = burnAmount.label === 0 ? '' : burnAmount.label;
            }}
            on:keyup={async () => {
              const debounced = debounce(burningAmountChange, 1000, {
                leading: false,
                trailing: true,
              });
              await debounced();
            }}
            bind:value={burnAmount.label}
            inputmode="decimal"
            title="Token Amount"
            autocomplete="off"
            autocorrect="off"
            type="number"
            pattern="^[0-9]*[.]?[0-9]*$"
            placeholder="0.0"
            minlength="1"
            maxlength="79"
            spellcheck="false"
          />
          <button class="swap-button">
            <span class="sc-iybRtq gjVeBU">
              <img
                class="h-auto w-24px mr-5px"
                alt={sellToken ? `${sellToken.symbol} logo` : ''}
                src={sellToken ? sellToken.icon : ''}
              />
              <span class="sc-kXeGPI jeVIZw token-symbol-container"
                >{sellToken ? sellToken.symbol : ''}</span
              >
            </span>
          </button>
        </div>
      </div>
      <button
        on:click={burn}
        disabled={burnAmount.label === 0 || burnAmount.label === '' || !$eth.address}
        class="stake-button rounded-20px p-15px w-100pc mt-auto"
      >
        Burn your DOUGH
      </button>
    </div>
  </div>
</div>
