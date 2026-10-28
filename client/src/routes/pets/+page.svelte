<script lang="ts">
	import { goto } from '$app/navigation';
	import { getPets } from '$lib/api/pet/PetController';
	import type { PetResponse } from '$lib/api/models';
	import { Input } from '$lib/components/ui/input';
	import * as Table from '$lib/components/ui/table';
	import { PawPrint, Search, Loader2, ChevronRight } from 'lucide-svelte';
	import { toast } from 'svelte-sonner';

	let pets = $state<PetResponse[]>([]);
	let loading = $state(true);
	let searchQuery = $state('');

	const filteredPets = $derived(
		pets.filter((pet) => pet.name.toLowerCase().includes(searchQuery.trim().toLowerCase()))
	);

	function petHref(pet: PetResponse): string {
		return `/pets/${pet.id}`;
	}

	async function loadPets() {
		loading = true;
		try {
			pets = await getPets();
		} catch (e) {
			toast.error('Failed to load pets');
			console.error('Error loading pets:', e);
		} finally {
			loading = false;
		}
	}

	$effect(() => {
		loadPets();
	});
</script>

<svelte:head>
	<title>Pets | VetHub</title>
</svelte:head>

<div class="container mx-auto px-4 py-8">
	<!-- Header -->
	<div class="mb-8">
		<div class="flex items-center gap-3 mb-2">
			<div class="flex h-10 w-10 items-center justify-center rounded-lg bg-primary/10">
				<PawPrint class="h-5 w-5 text-primary" />
			</div>
			<h1 class="text-3xl font-bold text-foreground">Pets</h1>
		</div>
		<p class="text-muted-foreground">Look up any patient registered at the clinic</p>
	</div>

	<!-- Search -->
	<div class="mb-6">
		<div class="relative max-w-md">
			<Search class="absolute left-3 top-1/2 h-4 w-4 -translate-y-1/2 text-muted-foreground" />
			<Input
				type="text"
				placeholder="Search by pet name..."
				class="pl-10"
				bind:value={searchQuery}
			/>
		</div>
	</div>

	<!-- Content -->
	{#if loading}
		<div class="flex items-center justify-center py-12">
			<Loader2 class="h-8 w-8 animate-spin text-primary" />
		</div>
	{:else if pets.length === 0}
		<div class="rounded-lg border border-dashed p-12 text-center">
			<PawPrint class="mx-auto h-12 w-12 text-muted-foreground/50" />
			<h3 class="mt-4 text-lg font-medium text-foreground">No pets yet</h3>
			<p class="mt-2 text-sm text-muted-foreground">
				Pets will appear here once they are registered for an owner.
			</p>
		</div>
	{:else}
		<div class="rounded-lg border bg-card">
			<Table.Root>
				<Table.Header>
					<Table.Row>
						<Table.Head>Name</Table.Head>
						<Table.Head>Type</Table.Head>
						<Table.Head>Birth date</Table.Head>
						<Table.Head>Owner</Table.Head>
					</Table.Row>
				</Table.Header>
				<Table.Body>
					{#each filteredPets as pet (pet.id)}
						<Table.Row class="cursor-pointer" onclick={() => goto(petHref(pet))}>
							<Table.Cell class="font-medium">
								<a
									href={petHref(pet)}
									class="inline-flex items-center gap-1 text-primary hover:underline"
								>
									{pet.name}
									<ChevronRight class="h-4 w-4" />
								</a>
							</Table.Cell>
							<Table.Cell>{pet.type?.name ?? '-'}</Table.Cell>
							<Table.Cell>{pet.birthDate}</Table.Cell>
							<Table.Cell>{pet.ownerFirstName} {pet.ownerLastName}</Table.Cell>
						</Table.Row>
					{:else}
						<Table.Row>
							<Table.Cell colspan={4} class="text-center text-muted-foreground py-8">
								No pets match your search
							</Table.Cell>
						</Table.Row>
					{/each}
				</Table.Body>
			</Table.Root>
		</div>

		<p class="mt-4 text-sm text-muted-foreground">
			Showing {filteredPets.length} of {pets.length} pets
		</p>
	{/if}
</div>
