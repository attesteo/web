<script lang="ts">
	import { ethers } from 'ethers';
	import { createPublicClient, http } from 'viem';
	import type { Address } from 'viem';
	import { optimism } from 'viem/chains';

	// Constants for Sign Protocol and OP token
	const SIGN_SCHEMA_UID = '0x0000000000000000000000000000000000000000000000000000000000000000'; // Replace with actual schema UID
	const OP_TOKEN_ADDRESS = '0x4200000000000000000000000000000000000042';
	const MIN_OP_REQUIRED = 1n; // Minimum OP tokens required to post

	const publicClient = createPublicClient({
		chain: optimism,
		transport: http()
	});

	interface Message {
		id: number;
		author: string;
		content: string;
		timestamp: Date;
		attestationUID: string | null;
	}

	let messages = $state<Message[]>([
		{
			id: 1,
			author: '0x1234...5678',
			content: 'Hello everyone!',
			timestamp: new Date('2024-03-20T10:00:00'),
			attestationUID: null
		}
	]);

	let newMessage = $state('');
	let walletAddress = $state<string | null>(null);
	let connecting = $state(false);
	let attesting = $state(false);

	async function checkOPBalance(address: string): Promise<boolean> {
		try {
			const balance = await publicClient.readContract({
				address: OP_TOKEN_ADDRESS as Address,
				abi: [
					{
						inputs: [{ name: 'account', type: 'address' }],
						name: 'balanceOf',
						outputs: [{ name: '', type: 'uint256' }],
						stateMutability: 'view',
						type: 'function'
					}
				],
				functionName: 'balanceOf',
				args: [address as Address]
			});

			return balance >= MIN_OP_REQUIRED;
		} catch (error) {
			console.error('Error checking OP balance:', error);
			return false;
		}
	}

	async function createAttestation(address: string): Promise<string> {
		attesting = true;
		try {
			// This is a simplified example. You'll need to implement the actual Sign Protocol attestation
			// using their SDK and your specific schema
			const response = await fetch('https://sign.global/api/v1/attestations', {
				method: 'POST',
				headers: {
					'Content-Type': 'application/json'
				},
				body: JSON.stringify({
					schema: SIGN_SCHEMA_UID,
					recipient: address,
					data: {
						tokenAddress: OP_TOKEN_ADDRESS,
						chainId: optimism.id,
						minBalance: MIN_OP_REQUIRED.toString()
					}
				})
			});

			if (!response.ok) {
				throw new Error('Failed to create attestation');
			}

			const data = await response.json();
			return data.uid;
		} finally {
			attesting = false;
		}
	}

	async function connectWallet() {
		if (!window.ethereum) {
			alert('Please install MetaMask to use this feature!');
			return;
		}

		connecting = true;
		try {
			const provider = new ethers.BrowserProvider(window.ethereum);
			const accounts = await provider.send('eth_requestAccounts', []);
			walletAddress = accounts[0];

			if (!walletAddress) {
				alert('Failed to connect wallet. Please try again.');
				return;
			}

			// Check OP token balance
			const hasEnoughOP = await checkOPBalance(walletAddress);
			if (!hasEnoughOP) {
				alert('You need at least 1 OP token to participate in the discussion.');
				walletAddress = null;
				return;
			}
		} catch (error) {
			console.error('Failed to connect wallet:', error);
			alert('Failed to connect wallet. Please try again.');
		} finally {
			connecting = false;
		}
	}

	function formatAddress(address: string): string {
		return `${address.slice(0, 6)}...${address.slice(-4)}`;
	}

	async function handleSubmit(e: Event) {
		e.preventDefault();
		if (!newMessage.trim() || !walletAddress) return;

		try {
			// Create attestation before posting
			const attestationUID = await createAttestation(walletAddress);

			if (!attestationUID) {
				alert('Failed to create attestation. Please try again.');
				return;
			}

			messages = [
				...messages,
				{
					id: messages.length + 1,
					author: formatAddress(walletAddress),
					content: newMessage,
					timestamp: new Date(),
					attestationUID
				}
			];

			newMessage = '';
		} catch (error) {
			console.error('Failed to create attestation:', error);
			alert('Failed to verify token ownership. Please try again.');
		}
	}
</script>

<div class="mx-auto max-w-3xl p-6">
	<div class="mb-8">
		<h1 class="mb-2 text-3xl font-bold">OP Grants Council Discussion</h1>
		<p class="text-gray-600">
			Join the conversation about Optimism's grant proposals and initiatives
		</p>
		<p class="mt-2 text-sm text-gray-500">
			* Requires at least 1 OP token on Optimism to participate in discussions
		</p>
	</div>

	<div class="mb-8 space-y-4">
		{#each messages as message}
			<div class="rounded-lg bg-white p-4 shadow">
				<div class="mb-2 flex items-start justify-between">
					<span class="font-semibold">{message.author}</span>
					<span class="text-sm text-gray-500">
						{message.timestamp.toLocaleString()}
					</span>
				</div>
				<p class="text-gray-700">{message.content}</p>
				{#if message.attestationUID}
					<div class="mt-2 text-xs text-gray-400">Verified via Sign Protocol</div>
				{/if}
			</div>
		{/each}
	</div>

	{#if !walletAddress}
		<div class="mb-8 text-center">
			<button
				onclick={connectWallet}
				disabled={connecting}
				class="rounded-md bg-blue-500 px-6 py-3 text-white transition-colors hover:bg-blue-600 disabled:bg-blue-300"
			>
				{connecting ? 'Connecting...' : 'Connect Wallet to Post'}
			</button>
		</div>
	{:else}
		<div class="mb-4 flex items-center justify-between rounded-lg bg-gray-50 p-4">
			<span class="text-sm">
				Connected as: <span class="font-medium">{formatAddress(walletAddress)}</span>
			</span>
			<button
				onclick={() => (walletAddress = null)}
				class="text-sm text-gray-500 hover:text-gray-700"
			>
				Disconnect
			</button>
		</div>

		<form onsubmit={handleSubmit} class="space-y-4">
			<div>
				<label for="message" class="mb-1 block text-sm font-medium text-gray-700"> Message </label>
				<textarea
					id="message"
					bind:value={newMessage}
					class="w-full rounded-md border border-gray-300 px-3 py-2"
					rows="3"
					placeholder="Type your message here..."
				></textarea>
			</div>

			<button
				type="submit"
				disabled={attesting}
				class="rounded-md bg-blue-500 px-4 py-2 text-white transition-colors hover:bg-blue-600 disabled:bg-blue-300"
			>
				{attesting ? 'Verifying...' : 'Send Message'}
			</button>
		</form>
	{/if}
</div>
