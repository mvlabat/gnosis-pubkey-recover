<script lang="ts">
  import { onMount } from 'svelte';
  import type { FormEventHandler } from 'svelte/elements';
  import { ethers } from 'ethers';
  import { clipboard } from '@skeletonlabs/skeleton';
  import { decodeTxData, packTransaction, parseSignatures, recoverAddresses } from '$lib/utils.js';

  const txHashRegex = /^0x([A-Fa-f0-9]{64})$/;

  let providerIsInitialized = false;
  let provider: ethers.BrowserProvider | ethers.AbstractProvider | undefined;
  let txHash: string = '';
  let addresses: { address: string; ensName: string | null }[] = [];
  let from = '';

  onMount(() => {
    if (window.ethereum) {
      provider = new ethers.BrowserProvider(window.ethereum);
    }
    providerIsInitialized = true;
  });

  const handleTxChange: FormEventHandler<HTMLInputElement> = async (event) => {
    if (!provider) {
      throw new Error("Ethereum provider isn't initialized");
    }

    if (!txHashRegex.test(event.currentTarget.value)) {
      console.warn('Invalid tx hash');
      return;
    }

    txHash = event.currentTarget.value;
    const tx = await provider.getTransaction(txHash);
    if (!tx) {
      console.warn('Failed to get the tx');
      return;
    }
    let data;
    try {
      data = decodeTxData(tx.data);
    } catch (e) {
      console.error(e);
      return;
    }

    const gnosisContract = new ethers.Contract(
      tx.to ?? '',
      [
        {
          inputs: [],
          name: 'nonce',
          outputs: [
            {
              internalType: 'uint256',
              name: '',
              type: 'uint256'
            }
          ],
          stateMutability: 'view',
          type: 'function'
        }
      ],
      provider
    );

    const nonce = await gnosisContract.nonce({ blockTag: (tx.blockNumber ?? 0) - 1 });

    const packedTx = packTransaction(data, nonce);
    const domainSeparator = '0x8554d768e75c4343acced7019fb5bdd54228945a3e9022412f42c25d46b88f69';
    const dataHash = ethers.solidityPackedKeccak256(
      ['bytes1', 'bytes1', 'uint256', 'uint256'],
      ['0x19', '0x01', domainSeparator, ethers.keccak256(packedTx)]
    );

    addresses = await Promise.all(
      recoverAddresses(dataHash, parseSignatures(data.signatures)).map(async (address) => ({
        address,
        ensName: await provider!.lookupAddress(address)
      }))
    );

    from = tx.from;
  };
</script>

<div>
  {#if providerIsInitialized && !provider}
    Web3 wallet isn't detected
  {/if}
</div>

<header class="card-header p-4">
  <h3 class="h3">Look up transaction signers</h3>
  <!-- svelte-ignore a11y-autofocus -->
  <input
    class="input mt-2.5"
    type="text"
    placeholder="0x..."
    on:input={handleTxChange}
    value={txHash}
    autofocus
  />
</header>

{#if addresses.length}
  <section class="p-4">
    <h3 class="h3">Transaction sender</h3>
    <a href="https://etherscan.io/address/{from}"><span class="code">{from}  <i class="fa fa-external-link" aria-hidden="true"></i></span></a>
    <button type="button" use:clipboard={from}>
      <i class="fa fa-copy" aria-hidden="true"></i>
    </button>
  </section>

  <section class="p-4">
    <h3 class="h3">Transaction signers</h3>
    {#each addresses as {address, ensName}}
      <div>
        <a target="_blank" href="https://etherscan.io/address/{address}">
          <span class="code">{address} <i class="fa fa-external-link" aria-hidden="true"></i></span>
        </a>
        <button type="button" use:clipboard={address}>
          <i class="fa fa-copy" aria-hidden="true"></i>
        </button>
        <a target="_blank" href="https://etherscan.io/address/{address}">
          {#if ensName}<span class="badge variant-filled">{ensName}</span>{/if}
        </a>
      </div>
    {/each}
  </section>
{/if}

<style>
</style>
