<script>
  import { onMount } from 'svelte';
  import { getAddress } from '@ethersproject/address';
  import { CloudflareProvider } from '@ethersproject/providers';
  import { setDefaults as setToast, toast } from 'bulma-toast';

  let input = null;
  let faucetInfo = {
    account: '0x0000000000000000000000000000000000000000',
    network: 'devnet',
    payout: 1,
    symbol: 'VLDM',
    hcaptcha_sitekey: '',
  };

  let mounted = false;
  let hcaptchaLoaded = false;

  onMount(async () => {
    const res = await fetch('/api/info');
    faucetInfo = await res.json();
    mounted = true;
  });

  window.hcaptchaOnLoad = () => {
    hcaptchaLoaded = true;
  };

  $: document.title = `${faucetInfo.symbol} ${capitalize(
    faucetInfo.network,
  )} Faucet`;

  let widgetID;
  $: if (mounted && hcaptchaLoaded) {
    widgetID = window.hcaptcha.render('hcaptcha', {
      sitekey: faucetInfo.hcaptcha_sitekey,
    });
  }

  setToast({
    position: 'bottom-center',
    dismissible: true,
    pauseOnHover: true,
    closeOnClick: false,
    animate: { in: 'fadeIn', out: 'fadeOut' },
    message: '',
  });

  async function handleRequest() {
    let address = input;
    if (address === null) {
      toast({ message: 'input required', type: 'is-warning' });
      return;
    }

    if (address.endsWith('.eth')) {
      try {
        const provider = new CloudflareProvider();
        address = await provider.resolveName(address);
        if (!address) {
          toast({ message: 'invalid ENS name', type: 'is-warning' });
          return;
        }
      } catch (error) {
        toast({ message: error.reason, type: 'is-warning' });
        return;
      }
    }

    try {
      address = getAddress(address);
    } catch (error) {
      toast({ message: error.reason, type: 'is-warning' });
      return;
    }

    try {
      let headers = {
        'Content-Type': 'application/json',
      };

      if (hcaptchaLoaded) {
        const { response } = await window.hcaptcha.execute(widgetID, {
          async: true,
        });
        headers['h-captcha-response'] = response;
      }

      const res = await fetch('/api/claim', {
        method: 'POST',
        headers,
        body: JSON.stringify({
          address,
        }),
      });

      let { msg } = await res.json();
      toast({ message: msg, type: res.ok ? 'is-success' : 'is-warning' });
    } catch (err) {
      console.error('Error requesting token: ', err);
      toast({ message: err.message, type: 'is-danger' });
    }
  }

  function capitalize(str) {
    const lower = str.toLowerCase();
    return str.charAt(0).toUpperCase() + lower.slice(1);
  }
</script>

<svelte:head>
  {#if mounted && faucetInfo.hcaptcha_sitekey}
    <script
      src="https://hcaptcha.com/1/api.js?onload=hcaptchaOnLoad&render=explicit"
      async
      defer
    ></script>
  {/if}
</svelte:head>

<main>
  <section class="hero is-info is-fullheight">
    <!-- Radial gradient -->
    <div class="blueRadialGradient00"></div>
    <div class="redRadialGradient00"></div>
    <!--  -->
    <div class="hero-head">
      <nav class="navbar">
        <div class="container">
          <div class="navbar-brand">
            <a href="https://www.validium.network/" class="brand">
              <div class="vldm-logo">
                <img src="./VLDM-Faucet-logo.png" alt="VLDM logo" />
              </div>
            </a>
          </div>
          <!-- <div id="navbarMenu" class="navbar-menu">
            <div class="navbar-end">
              <span class="navbar-item">
                <a
                  class="button is-white is-outlined"
                  href="https://github.com/chainflag/eth-faucet"
                >
                  <span class="icon">
                    <i class="fa fa-github" />
                  </span>
                  <span>View Source</span>
                </a>
              </span>
            </div>
          </div> -->
        </div>
      </nav>
    </div>

    <div class=" main-section">
      <div class="container has-text-centered">
        <div class="column is-8 is-offset-2 content-wrapper">
          <div class="heading-wrapper">
            <h1 class="h1">
              Receive
              <span class="text-primary">
                {faucetInfo.payout}
                {faucetInfo.symbol}
              </span>
              per request
            </h1>
            <p class="p">
              Use this faucet to get VLDM test funds. Ensure your wallet is
              connected to the Validium network.
            </p>
          </div>
          <div id="hcaptcha" data-size="invisible"></div>
          <h2 class="h2">
            Serving from <br />
            {faucetInfo.account}
          </h2>
          <div class="box">
            <div class="field is-grouped">
              <p class="control is-expanded">
                <input
                  bind:value={input}
                  class="address-input"
                  type="text"
                  placeholder="Enter your address or ENS name"
                />
              </p>
              <p class="control">
                <button on:click={handleRequest} class="request-btn">
                  Request
                </button>
              </p>
            </div>
          </div>
        </div>
      </div>
    </div>
  </section>
</main>

<style>
  .hero.is-info {
    background-color: #180f2d;
    position: relative;
    overflow: hidden;
  }
  .main-section {
    /* border: 1px solid red; */
    flex-grow: 1;
    z-index: 20;
  }
  .box {
    border: 2px solid rgba(48, 39, 100, 0.58);
    background-color: #181235;
    padding: 2rem 1.5rem;
  }
  .text-primary {
    color: #d10045;
  }
  .brand {
    display: flex;
    align-items: center;
    gap: 0.5rem;
    /* border: 2px solid orange; */
    padding: 1.5rem;
  }
  .vldm-logo {
    width: 10rem;
  }
  .address-input {
    padding: 0.75rem 1.25rem;
    width: 100%;
    flex: 1 1 200px;
    border-radius: 10px;
    border: none;
    background-color: #201a3e;
    font-size: 1rem;
    color: white;

    &::placeholder {
      color: #b9b9df;
    }
  }
  .request-btn {
    padding: 0.75rem 2rem;
    border-radius: 30px;
    font-size: 1rem;
    font-weight: 600;
    border: none;
    background-color: #d10045;
    color: white;
    cursor: pointer;
  }
  .blueRadialGradient00 {
    position: absolute;
    top: -65rem;
    left: -45.5rem;
    filter: blur(64px);
    width: 85rem;
    height: 85rem;
    background: radial-gradient(ellipse, #7a42d6df 0%, transparent 80%);
    z-index: 10;
  }
  .redRadialGradient00 {
    position: absolute;
    top: 30rem;
    right: -8rem;
    filter: blur(64px);
    width: 22.5rem;
    height: 22.5rem;
    background: radial-gradient(ellipse, #d1004521 0%, transparent 100%);
    z-index: 10;
  }
  .content-wrapper {
    /* border: 2px solid pink; */
    min-height: 80vh;
    display: flex;
    flex-direction: column;
    justify-content: center;
    gap: 2rem;
  }
  .heading-wrapper {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 1.5rem;
  }
  .h1 {
    font-size: 3rem;
    font-weight: 700;
    line-height: 3rem;
  }
  .h2 {
    font-size: 1.25rem;
    font-weight: 400;
    line-height: 1.5rem;
    word-break: break-all;
  }
  .p {
    font-size: 1rem;
    font-weight: 500;
    line-height: 1.125rem;
    max-width: 26rem;
  }
</style>
