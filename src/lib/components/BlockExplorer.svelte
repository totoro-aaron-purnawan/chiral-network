<script lang="ts">
  import { onMount } from 'svelte';
  import { invoke } from '@tauri-apps/api/core';

  interface BlockDetail {
    number: number;
    hash: string;
    miner: string;
    timestamp: number;
    transaction_count: number;
    transactions: any[];
  }

  interface TransactionDetail {
    hash: string;
    from: string;
    to: string;
    value: string;
    gas: string;
    gasPrice: string;
  }

  let blocks: BlockDetail[] = [];
  let error: string | null = null;
  let offset = 0;
  const limit = 10;
  let currentHeight = 0;
  let selectedBlock: BlockDetail | null = null;
  let transactionHash = '';
  let transactionResult: TransactionDetail | null = null;
  let addressBalance = '';
  let balanceResult: string | null = null;
  let balanceError: string | null = null;
  let isLoading = false;
  let isTauri = false;

  async function fetchBlocks() {
    if (!isTauri) {
      error = 'Block explorer requires Tauri desktop app. Please run "npm run tauri dev" instead of "npm run dev".';
      isLoading = false;
      return;
    }

    try {
      isLoading = true;
      error = null;
      
      // Add timeout to prevent hanging
      const timeoutPromise = new Promise((_, reject) => 
        setTimeout(() => reject(new Error('Request timeout')), 10000)
      );
      
      const fetchPromise = Promise.all([
        invoke('get_latest_blocks', { limit, offset }) as Promise<BlockDetail[]>,
        invoke('get_current_block') as Promise<number>
      ]);
      
      const [fetchedBlocks, height] = await Promise.race([fetchPromise, timeoutPromise]) as [BlockDetail[], number];
      blocks = fetchedBlocks;
      currentHeight = height;
    } catch (e) {
      error = `Failed to fetch blockchain data. Please start the Geth node on the <a href="/network" class="text-blue-300 hover:underline">Network page</a>.`;
      // Show some placeholder data instead of empty state
      if (blocks.length === 0) {
        blocks = [{
          number: 0,
          hash: '0x0000000000000000000000000000000000000000000000000000000000000000',
          miner: '0x0000000000000000000000000000000000000000',
          timestamp: Math.floor(Date.now() / 1000),
          transaction_count: 0,
          transactions: []
        }];
        currentHeight = 0;
      }
    } finally {
      isLoading = false;
    }
  }

  async function fetchTransaction() {
    if (!transactionHash.trim()) return;
    
    if (!isTauri) {
      balanceError = 'Transaction lookup requires Tauri desktop app.';
      return;
    }
    
    try {
      balanceError = null;
      // Note: This would need a corresponding Tauri command to be implemented
      // For now, we'll show a placeholder with the actual hash
      transactionResult = {
        hash: transactionHash,
        from: '0x0000000000000000000000000000000000000000',
        to: '0x0000000000000000000000000000000000000000',
        value: '0',
        gas: '21000',
        gasPrice: '20000000000'
      };
    } catch (e) {
      balanceError = e as string;
    }
  }

  async function fetchBalance() {
    if (!addressBalance.trim()) return;
    
    if (!isTauri) {
      balanceError = 'Balance check requires Tauri desktop app.';
      return;
    }
    
    try {
      balanceError = null;
      const balance = await invoke('get_account_balance', { address: addressBalance }) as string;
      balanceResult = `${balance} ETH`;
    } catch (e) {
      balanceError = e as string;
    }
  }

  function nextPage() {
    offset += limit;
    fetchBlocks();
  }

  function prevPage() {
    offset = Math.max(0, offset - limit);
    fetchBlocks();
  }

  function selectBlock(block: BlockDetail) {
    selectedBlock = block;
  }

  function closeBlockDetails() {
    selectedBlock = null;
  }

  onMount(() => {
    // Check if we're running in Tauri - more robust detection
    isTauri = typeof window !== 'undefined' && 
              (window.__TAURI__ !== undefined || 
               window.__TAURI_INTERNALS__ !== undefined ||
               (window as any).__TAURI_METADATA__ !== undefined);
    
    if (isTauri) {
      fetchBlocks();
      // Set up real-time updates every 10 seconds
      const interval = setInterval(fetchBlocks, 10000);
      return () => clearInterval(interval);
    } else {
      // Show error message for web browser
      error = 'Block explorer requires Tauri desktop app. Please run "npm run tauri dev" instead of "npm run dev".';
    }
  });
</script>

<div class="bg-gray-800 text-white p-6 rounded-lg">
  <div class="flex justify-between items-center mb-6">
    <h2 class="text-2xl font-bold">Block Explorer</h2>
    <div class="text-sm text-gray-300">
      Current Height: <span class="font-mono text-blue-400">{currentHeight.toLocaleString()}</span>
    </div>
  </div>

  {#if error}
    <div class="bg-red-900 border border-red-700 text-red-200 px-4 py-3 rounded mb-4">
      Error: {@html error}
    </div>
  {/if}

  <!-- Search and Lookup Section -->
  <div class="grid grid-cols-1 md:grid-cols-2 gap-4 mb-6">
    <!-- Transaction Lookup -->
    <div class="bg-gray-700 p-4 rounded-lg">
      <h3 class="text-lg font-semibold mb-3">Transaction Lookup</h3>
      <div class="flex gap-2">
        <input
          type="text"
          bind:value={transactionHash}
          placeholder="Enter transaction hash..."
          class="flex-1 px-3 py-2 bg-gray-600 border border-gray-500 rounded text-white placeholder-gray-400 focus:outline-none focus:border-blue-500"
        />
        <button
          on:click={fetchTransaction}
          disabled={!transactionHash.trim()}
          class="bg-blue-500 hover:bg-blue-600 disabled:opacity-50 disabled:cursor-not-allowed text-white px-4 py-2 rounded font-medium"
        >
          Lookup
        </button>
      </div>
      <p class="text-xs text-gray-400 mt-2">
        Note: Transaction lookup shows placeholder data. Backend implementation needed.
      </p>
      {#if transactionResult}
        <div class="mt-3 p-3 bg-gray-600 rounded text-sm">
          <div class="grid grid-cols-2 gap-2">
            <div><span class="text-gray-400">Hash:</span> <span class="font-mono text-xs break-all">{transactionResult.hash}</span></div>
            <div><span class="text-gray-400">From:</span> <span class="font-mono text-xs">{transactionResult.from}</span></div>
            <div><span class="text-gray-400">To:</span> <span class="font-mono text-xs">{transactionResult.to}</span></div>
            <div><span class="text-gray-400">Value:</span> <span class="font-mono text-xs">{transactionResult.value} ETH</span></div>
          </div>
        </div>
      {/if}
    </div>

    <!-- Address Balance Check -->
    <div class="bg-gray-700 p-4 rounded-lg">
      <h3 class="text-lg font-semibold mb-3">Address Balance</h3>
      <div class="flex gap-2">
        <input
          type="text"
          bind:value={addressBalance}
          placeholder="Enter address..."
          class="flex-1 px-3 py-2 bg-gray-600 border border-gray-500 rounded text-white placeholder-gray-400 focus:outline-none focus:border-blue-500"
        />
        <button
          on:click={fetchBalance}
          disabled={!addressBalance.trim()}
          class="bg-green-500 hover:bg-green-600 disabled:opacity-50 disabled:cursor-not-allowed text-white px-4 py-2 rounded font-medium"
        >
          Check
        </button>
      </div>
      {#if balanceResult}
        <div class="mt-3 p-3 bg-gray-600 rounded text-sm">
          <div class="text-center">
            <span class="text-gray-400">Balance:</span>
            <span class="font-mono text-lg text-green-400 ml-2">{balanceResult}</span>
          </div>
        </div>
      {/if}
      {#if balanceError}
        <div class="mt-3 p-3 bg-red-900 border border-red-700 rounded text-sm text-red-200">
          {balanceError}
        </div>
      {/if}
    </div>
  </div>

  <!-- Blocks Table -->
  <div class="bg-gray-700 p-4 rounded-lg">
    <div class="flex justify-between items-center mb-4">
      <h3 class="text-lg font-semibold">Latest Blocks</h3>
      <div class="flex gap-2">
        <button
          on:click={prevPage}
          disabled={offset === 0 || isLoading}
          class="bg-blue-500 hover:bg-blue-600 disabled:opacity-50 disabled:cursor-not-allowed text-white px-4 py-2 rounded font-medium"
        >
          Previous
        </button>
        <button
          on:click={nextPage}
          disabled={isLoading}
          class="bg-blue-500 hover:bg-blue-600 disabled:opacity-50 disabled:cursor-not-allowed text-white px-4 py-2 rounded font-medium"
        >
          Next
        </button>
      </div>
    </div>

    {#if isLoading}
      <div class="text-center py-8">
        <div class="inline-block animate-spin rounded-full h-8 w-8 border-b-2 border-blue-500"></div>
        <p class="mt-2 text-gray-400">Loading blocks...</p>
      </div>
    {:else}
      <div class="overflow-x-auto">
        <table class="min-w-full divide-y divide-gray-600">
          <thead class="bg-gray-600">
            <tr>
              <th class="px-6 py-3 text-left text-xs font-medium text-gray-300 uppercase tracking-wider">Block #</th>
              <th class="px-6 py-3 text-left text-xs font-medium text-gray-300 uppercase tracking-wider">Hash</th>
              <th class="px-6 py-3 text-left text-xs font-medium text-gray-300 uppercase tracking-wider">Miner</th>
              <th class="px-6 py-3 text-left text-xs font-medium text-gray-300 uppercase tracking-wider">Transactions</th>
              <th class="px-6 py-3 text-left text-xs font-medium text-gray-300 uppercase tracking-wider">Timestamp</th>
              <th class="px-6 py-3 text-left text-xs font-medium text-gray-300 uppercase tracking-wider">Actions</th>
            </tr>
          </thead>
          <tbody class="bg-gray-800 divide-y divide-gray-600">
            {#each blocks as block}
              <tr class="hover:bg-gray-700 transition-colors">
                <td class="px-6 py-4 whitespace-nowrap font-mono text-blue-400">{block.number}</td>
                <td class="px-6 py-4 whitespace-nowrap font-mono text-xs text-gray-300 truncate max-w-xs" title={block.hash}>
                  {block.hash}
                </td>
                <td class="px-6 py-4 whitespace-nowrap font-mono text-xs text-gray-300 truncate max-w-xs" title={block.miner}>
                  {block.miner}
                </td>
                <td class="px-6 py-4 whitespace-nowrap text-center">
                  <span class="bg-blue-600 text-white px-2 py-1 rounded-full text-xs">
                    {block.transaction_count}
                  </span>
                </td>
                <td class="px-6 py-4 whitespace-nowrap text-sm text-gray-300">
                  {new Date(block.timestamp * 1000).toLocaleString()}
                </td>
                <td class="px-6 py-4 whitespace-nowrap">
                  <button
                    on:click={() => selectBlock(block)}
                    class="bg-gray-600 hover:bg-gray-500 text-white px-3 py-1 rounded text-sm font-medium"
                  >
                    Details
                  </button>
                </td>
              </tr>
            {/each}
          </tbody>
        </table>
      </div>
    {/if}
  </div>
</div>

<!-- Block Details Modal -->
{#if selectedBlock}
  <div class="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center z-50 p-4">
    <div class="bg-gray-800 rounded-lg max-w-4xl w-full max-h-[90vh] overflow-y-auto">
      <div class="p-6">
        <div class="flex justify-between items-center mb-4">
          <h3 class="text-xl font-bold">Block #{selectedBlock.number} Details</h3>
          <button
            on:click={closeBlockDetails}
            class="text-gray-400 hover:text-white text-2xl font-bold"
          >
            ×
          </button>
        </div>
        
        <div class="grid grid-cols-1 md:grid-cols-2 gap-6">
          <div class="space-y-4">
            <div>
              <div class="text-sm font-medium text-gray-400 mb-1">Block Hash</div>
              <div class="p-3 bg-gray-700 rounded font-mono text-sm break-all">
                {selectedBlock.hash}
              </div>
            </div>
            
            <div>
              <div class="text-sm font-medium text-gray-400 mb-1">Miner Address</div>
              <div class="p-3 bg-gray-700 rounded font-mono text-sm break-all">
                {selectedBlock.miner}
              </div>
            </div>
            
            <div>
              <div class="text-sm font-medium text-gray-400 mb-1">Timestamp</div>
              <div class="p-3 bg-gray-700 rounded">
                {new Date(selectedBlock.timestamp * 1000).toLocaleString()}
              </div>
            </div>
          </div>
          
          <div class="space-y-4">
            <div>
              <div class="text-sm font-medium text-gray-400 mb-1">Transaction Count</div>
              <div class="p-3 bg-gray-700 rounded text-center">
                <span class="text-2xl font-bold text-blue-400">{selectedBlock.transaction_count}</span>
              </div>
            </div>
            
            <div>
              <div class="text-sm font-medium text-gray-400 mb-1">Block Number</div>
              <div class="p-3 bg-gray-700 rounded font-mono text-center">
                {selectedBlock.number}
              </div>
            </div>
          </div>
        </div>
        
        {#if selectedBlock.transactions && selectedBlock.transactions.length > 0}
          <div class="mt-6">
            <h4 class="text-lg font-semibold mb-3">Transactions</h4>
            <div class="space-y-2 max-h-60 overflow-y-auto">
              {#each selectedBlock.transactions as tx}
                <div class="p-3 bg-gray-700 rounded text-sm">
                  <div class="font-mono text-xs break-all">{tx.hash || 'N/A'}</div>
                </div>
              {/each}
            </div>
          </div>
        {/if}
      </div>
    </div>
  </div>
{/if}
''