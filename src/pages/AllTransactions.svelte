<script lang="ts">
  import { onMount } from 'svelte';
  import { invoke } from '@tauri-apps/api/tauri';
  import { Button } from '$lib/components/ui/button';
  import { toast } from 'svelte-sonner';

  // Hardcode Chain ID as requested for this temporary page
  let chainId: number | string = 777; // From genesis.json
  let transactions: any[] = [];
  let isLoading = false;
  let error: string | null = null;

  async function fetchAllTransactions() {
    isLoading = true;
    error = null;
    try {
      const result = await invoke('get_all_transactions');
      transactions = result as any[];
      if (transactions.length === 0) {
        toast.info('No transactions found on the blockchain.');
      }
    } catch (e) {
      console.error('Failed to fetch all transactions:', e);
      error = `Failed to fetch transactions: ${e}`;
      toast.error(error);
    } finally {
      isLoading = false;
    }
  }

  // No need to call fetchChainId onMount anymore as it's hardcoded
</script>

<div class="p-4 md:p-6">
  <div class="flex justify-between items-center mb-4">
    <div>
      <h1 class="text-3xl font-bold">All Blockchain Transactions</h1>
      <p class="text-muted-foreground">A temporary tool to inspect all transactions directly from the chain.</p>
    </div>
    <div class="text-right">
        <p class="text-sm text-muted-foreground">Chain ID</p>
        <p class="text-lg font-bold">{chainId}</p>
    </div>
  </div>

  <div class="mb-6">
    <Button on:click={fetchAllTransactions} disabled={isLoading}>
      {isLoading ? 'Loading...' : 'Fetch All Transactions'}
    </Button>
  </div>

  {#if error}
    <div class="bg-red-100 border border-red-400 text-red-700 px-4 py-3 rounded relative" role="alert">
      <strong class="font-bold">Error: </strong>
      <span class="block sm:inline">{error}</span>
    </div>
  {/if}

  <div class="space-y-4">
    {#if transactions.length > 0}
      {#each transactions as tx, i (tx.hash)}
        <div class="bg-card border rounded-lg p-4">
          <div class="flex justify-between items-start">
            <p class="font-mono text-sm break-all"><strong>Hash:</strong> {tx.hash}</p>
            <p class="text-sm whitespace-nowrap ml-4"><strong>Block:</strong> {tx.blockNumber}</p>
          </div>
          <hr class="my-2"/>
          <div class="grid grid-cols-1 md:grid-cols-2 gap-x-4 gap-y-2 text-sm">
            <div><strong>From:</strong> <span class="font-mono break-all">{tx.from}</span></div>
            <div><strong>To:</strong> <span class="font-mono break-all">{tx.to || 'N/A (Contract Creation?)'}</span></div>
            <div><strong>Value:</strong> <span>{(parseInt(tx.value, 16) / 1e18).toFixed(8)} ETH</span></div>
            <div><strong>Gas Price:</strong> <span>{parseInt(tx.gasPrice, 16)} wei</span></div>
            <div><strong>Gas Used:</strong> <span>{tx.gas}</span></div>
            <div><strong>Nonce:</strong> <span>{parseInt(tx.nonce, 16)}</span></div>
            <div><strong>Input Data:</strong> <span class="font-mono break-all text-xs">{tx.input || '0x'}</span></div>
            <div><strong>Transaction Index:</strong> <span>{parseInt(tx.transactionIndex, 16)}</span></div>
          </div>
        </div>
      {/each}
    {:else if !isLoading}
      <div class="text-center py-8 text-muted-foreground">
        <p>No transactions to display. Click the button to fetch them.</p>
      </div>
    {/if}
  </div>
</div>