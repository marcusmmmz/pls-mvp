<script lang="ts">
	import Person from '$lib/components/Person.svelte';
	import { Button, P } from 'flowbite-svelte';
	import { getContext } from 'svelte';
	import type { Writable } from 'svelte/store';
	import type { NewContractData } from '../shared';
	import { broadcastToNostr, makeNostrEvent, nostrEncryptDmFactory } from '$lib/nostr';
	import { goto } from '$app/navigation';
	import { ContractRequestEvent, type ContractRequestPayload } from '../../shared';
	import { isValidNetworkName } from '$lib/bitcoin';
	import { generateSecretKey, getPublicKey } from 'nostr-tools';
	import { tweakContractPubkey } from '$lib/pls/contract';

	let newContract = getContext<Writable<NewContractData>>('contract');

	$: clients = $newContract.clients!;
	$: arbitrators = $newContract.arbitrators!;
	$: arbitratorsQuorum = $newContract.arbitratorsQuorum!;
	$: documentName = $newContract.fileName!;
	$: documentHash = $newContract.fileHash!;
	$: network = $newContract.network!;

	async function requestSignatures() {
		if (!clients[0] || !clients[1]) return alert('Contracts need exactly 2 clients');

		if (arbitrators.length === 0) return alert('Select at least one arbitrator');

		if (!documentHash) return alert('Please select a file')!;

		const pubkeys = [clients[0], clients[1], ...arbitrators].map((pub) => pub.slice(-64));

		if (!isValidNetworkName(network)) return alert('Invalid network');

		const contractPrivkey = generateSecretKey();
		const contractPrivkeyStr = Buffer.from(contractPrivkey).toString('hex');
		const contractPubkey = getPublicKey(contractPrivkey);
		const contractEncryptDm = nostrEncryptDmFactory(contractPrivkeyStr);

		const contractRequestPayload = {
			arbitratorPubkeys: arbitrators.map((pub) => pub.slice(-64)),
			arbitratorsQuorum,
			clientPubkeys: [clients[0], clients[1]].map((pub) => pub.slice(-64)),
			fileHash: documentHash,
			network
		} satisfies ContractRequestPayload;

		const contractRequestEncryptedText = await contractEncryptDm.encryptDM(
			contractPubkey,
			JSON.stringify(contractRequestPayload)
		);

		const contractPrivkeySecretsTable = await Promise.all(
			pubkeys.map(async (pubkey) => {
				const encryptedSecret = await contractEncryptDm.encryptDM(
					pubkey,
					Buffer.from(contractPrivkey).toString('hex')
				);

				const tweakedPubkey = tweakContractPubkey(documentHash, pubkey);

				return ['secret', tweakedPubkey, encryptedSecret];
			})
		);

		const pubkeysHashTable = pubkeys.map((pubkey) => [
			'h',
			tweakContractPubkey(documentHash, pubkey)
		]);

		const contractRequestEvent = await makeNostrEvent(
			contractPrivkeyStr,
			ContractRequestEvent,
			contractRequestEncryptedText,
			[...pubkeysHashTable, ...contractPrivkeySecretsTable]
		);

		const contractEventId = contractRequestEvent.id;

		broadcastToNostr(contractRequestEvent);

		goto('/contract/join?eventId=' + contractEventId);
	}
</script>

<P class="text-2xl">Review</P>

<div class="flex flex-col justify-center items-center gap-2 h-full">
	<P size="xl" weight="bold">Clients:</P>
	{#if clients[0] || clients[1]}
		<div class="flex gap-2">
			<Person pubkey={clients[0] ?? ''} />
			<Person pubkey={clients[1] ?? ''} />
		</div>
	{/if}
	<P size="xl" weight="bold">Arbitrators:</P>
	<div class="flex gap-2 flex-wrap justify-center">
		{#each arbitrators as arbitrator}
			<Person pubkey={arbitrator ?? ''} />
		{/each}
	</div>
	<P size="xl">Arbitrators quorum: <strong>{arbitratorsQuorum}</strong></P>
	<P size="xl">Network: <strong>{network}</strong></P>
	<P size="xl">File: <strong>{documentName}</strong></P>
</div>
<Button class="w-48 md:w-52" on:click={requestSignatures}>Create</Button>
