<script>
  import { writable } from 'svelte/store';
  import Web3 from 'web3';
  import { ABI } from './lib/ContractABI';

  let web3;
  let errorMessage = writable('');
  let outputMessage = writable('');
  const account = writable('');
  const contractAddress = writable('0x2b591e99afe9f32eaa6214f7b7629768c40eeb39'); // Default or previous known address
  const contractABI = writable(JSON.stringify(ABI, null, 2));
  const contract = writable(null);
  const readFunctions = writable([]);
  const writeFunctions = writable([]);
  const activeTab = writable('read');

  async function connectWallet() {
    if (window.ethereum) {
      web3 = new Web3(window.ethereum);
      try {
        const accounts = await window.ethereum.request({ method: 'eth_requestAccounts' });
        account.set(accounts[0]);
        initializeContract();
      } catch (error) {
        errorMessage.set(`Failed to connect to MetaMask: ${error.message}`);
        console.error(error);
      }
    } else {
      errorMessage.set("Non-Ethereum browser detected. You should consider trying MetaMask!");
    }
  }

  function initializeContract() {
    try {
      const trimmedABI = $contractABI.trim();
      const parsedABI = JSON.parse(trimmedABI);
      contract.set(new web3.eth.Contract(parsedABI, $contractAddress));
      setFunctionLists(parsedABI);
      errorMessage.set('');
    } catch (error) {
      errorMessage.set(`Error parsing ABI: ${error.message}`);
      console.error(`Error parsing ABI: ${error.message}`);
    }
  }

  function setFunctionLists(abi) {
    const readFuncs = abi.filter(item => item.type === 'function' && (item.stateMutability === 'view' || item.stateMutability === 'pure'));
    const writeFuncs = abi.filter(item => item.type === 'function' && (item.stateMutability === 'nonpayable' || item.stateMutability === 'payable'));
    readFunctions.set(readFuncs.map(func => ({
      name: func.name,
      inputs: func.inputs.map(input => ({ type: input.type, name: input.name, value: '' }))
    })));
    writeFunctions.set(writeFuncs.map(func => ({
      name: func.name,
      inputs: func.inputs.map(input => ({ type: input.type, name: input.name, value: '' }))
    })));
  }

  async function callFunction(funcName, args) {
    const method = $contract.methods[funcName](...args);
    const funcObj = $contract.options.jsonInterface.find(f => f.name === funcName && f.type === 'function');
    if (funcObj && ['pure', 'view'].includes(funcObj.stateMutability)) {
      method.call().then(result => {
        outputMessage.set(`${result}`);
      }).catch(err => {
        errorMessage.set(`Error on read call for ${funcName}: ${err.message}`);
      });
    } else if (funcObj) {
      method.send({ from: $account }).then(result => {
        outputMessage.set(`${result}`);
      }).catch(err => {
        errorMessage.set(`Error on write transaction for ${funcName}: ${err.message}`);
      });
    } else {
      errorMessage.set("Function information not found in ABI");
    }
  }

  function checkConnectedAccount() {
    if (window.ethereum) {
      web3 = new Web3(window.ethereum);
      web3.eth.getAccounts().then(accounts => {
        if (accounts.length > 0) {
          account.set(accounts[0]);
          initializeContract();
        }
      }).catch(error => {
        errorMessage.set(`Error checking existing accounts: ${error.message}`);
      });
    }
  }

  checkConnectedAccount();
</script>

<main>
  <h1>MetaMask Connection</h1>
  {#if $errorMessage}
    <p style="color: red">{$errorMessage}</p>
  {/if}

  {#if $account}
    <div class="outputDiv">
      <h1>Output: {$outputMessage}</h1>
    </div>

    <h2>Connected Account</h2>
    <p>{$account}</p>
    
    <div class="contract-container">
      <input type="text" placeholder="Contract Address" bind:value={$contractAddress} />
      <textarea class="text-area" placeholder="Contract ABI" bind:value={$contractABI}></textarea>
      <button on:click={initializeContract}>Update Contract Info</button>
    </div>

    <br>
    <br>

    <nav>
      <button on:click={() => activeTab.set('read')} class:active={$activeTab === 'read'}>Read Functions</button>
      <button on:click={() => activeTab.set('write')} class:active={$activeTab === 'write'}>Write Functions</button>
    </nav>

    <br>
    <br>

    {#if $activeTab === 'read'}
      <h3>Read Functions</h3>
      <ul>
        {#each $readFunctions as func}
          <li>
            {func.name}
            {#each func.inputs as input}
              <input placeholder={input.name + " (" + input.type + ")"} bind:value={input.value} />
            {/each}
            <button on:click={() => callFunction(func.name, func.inputs.map(i => i.value))}>Call</button>
          </li>
        {/each}
      </ul>
    {:else if $activeTab === 'write'}
      <h3>Write Functions</h3>
      <ul>
        {#each $writeFunctions as func}
          <li>
            {func.name}
            {#each func.inputs as input}
              <input placeholder={input.name + " (" + input.type + ")"} bind:value={input.value} />
            {/each}
            <button on:click={() => callFunction(func.name, func.inputs.map(i => i.value))}>Send Transaction</button>
          </li>
        {/each}
      </ul>
    {/if}
  {:else}
    <button on:click={connectWallet}>Connect Wallet</button>
  {/if}
</main>

<style>
  main {
    text-align: center;
    padding: 1em;
    max-width: 1500px; /* Increased width for better layout */
    margin: auto;
    background: #f9f9f9;
    border-radius: 8px;
    box-shadow: 0 0 10px rgba(0, 0, 0, 0.1);
  }

  ul {
    list-style-type: none; /* Remove bullets */
    padding: 0;
  }

  li {
    margin-bottom: 1em; /* Space between items */
  }

  input {
    margin-right: 0.5em; /* Space before the button */
    padding: 0.5em;
    border: 1px solid #ccc;
    border-radius: 4px;
  }

  .contract-container {
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 0.5em; /* Space between elements */
    margin-bottom: 1em; /* Space below the container */
  }

  .contract-container input {
    flex: 1;
  }

  .contract-container .text-area {
    flex: 2;
    height: 55px; /* Sets a default height */
    padding: 0.5em;
    border: 1px solid #ccc;
    border-radius: 4px;
  }

  button {
    flex: 1;
    padding: 0.5em 1em;
    font-size: 1em;
    background-color: #007BFF;
    color: white;
    border: none;
    border-radius: 4px;
    cursor: pointer;
  }

  .contract-container button {
    flex: 1;
    padding: 0.5em 1em;
    font-size: 1em;
    background-color: #007BFF;
    color: white;
    border: none;
    border-radius: 4px;
    cursor: pointer;
  }

  button:hover {
    background-color: #0056b3;
  }

  nav button {
    margin: 0 5px;
    background-color: #f0f0f0;
    color: black;
  }

  nav .active {
    font-weight: bold;
    background-color: #007BFF;
    color: white;
  }

  p {
    color: #d00; /* More generic error styling */
  }
</style>