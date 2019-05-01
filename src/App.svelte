<header>
	<nav class="navbar shadow-sm">
		<a
		 class="navbar-brand text-white"
		 href="/"
		>French address navigator</a>
	</nav>
</header>
<svelte:window on:resize={resize} />
<main
 role="main"
 class="container-fluid my-4"
>
	<div class="row justify-content-center">
		<div class="col-sm-12 col-md-8 col-lg-6">
			<form>
				<div class="form-group">
					<div class="input-group shadow-sm">
						<input
						 bind:value={address}
						 on:input="{e => getAddresses(e.target.value)}"
						 class="form-control form-control-lg"
						 type="text"
						 placeholder="Type the address"
						/>
						<div class="input-group-append">
							<span class="input-group-text">
								<i
								 aria-hidden="true"
								 class="material-icons"
								> search </i>
							</span>
						</div>
					</div>
					{#if addressLoading}
					<div class="progress">
						<div
						 class="progress-bar progress-bar-striped progress-bar-animated"
						 role="progressbar"
						 aria-valuenow="75"
						 aria-valuemin="0"
						 aria-valuemax="100"
						 style="width: 100%"
						></div>
					</div>
					{/if}
				</div>
			</form>
		</div>
	</div>
	<div class="row justify-content-center">
		<div
		 class="col-sm-12 col-md-8 col-lg-6"
		 bind:this={addressTableRow}
		>
			<table class="table table-bordered shadow-sm">
				<thead>
					<tr>
						<th scope="col">City</th>
						<th
						 style="cursor: pointer;"
						 scope="col"
						 on:click={setNameSortAscending}
						>
							Name
							<i
							 class="material-icons"
							 style="font-size: 16px; width: 16px; height: 16pxd;"
							>
								{#if nameSortAscending != null} {#if nameSortAscending} arrow_upward {:else}
								arrow_downward {/if} {/if}
							</i>
						</th>
					</tr>
				</thead>
				<tbody>
					{#each sortedByNameSearchedAddresses as sa}
					<tr>
						<td>{sa.properties.city}</td>
						<td>{sa.properties.name}</td>
					</tr>
					{:else}
					<tr>
						<td
						 colspan="2"
						 class="text-center"
						>No data available</td>
					</tr>
					{/each}
				</tbody>
			</table>
		</div>
	</div>
	<div class="row justify-content-center">
		<div class="col-sm-12 col-md-8 col-lg-6">
			<div
			 bind:this={mapContainer}
			 class="shadow-sm"
			 style="height: {mapSize}px;"
			>
				<!-- map is rendered here -->
			</div>
		</div>
	</div>
</main>
<footer class="footer shadow-sm">
	<div class="container-fluid">
		<div class="row">
			<div class="col">
				<span class="text-white">Made with love by Yuriy © 2019 - Eternity.</span>
			</div>
		</div>
	</div>
</footer>

<script>
	import axios from "axios";
	import { onMount } from "svelte";
	import GoogleMapsLoader from "google-maps";

	const sortDirectionMap = {
	  true: false,
	  false: null,
	  null: true
	};
	const calculateWidth = element => {
	  const computedStyle = getComputedStyle(element);

	  const elementWidth =
	    element.clientWidth -
	    parseFloat(computedStyle.paddingLeft) -
	    parseFloat(computedStyle.paddingRight);

	  return elementWidth;
	};

	onMount(() => {
	  GoogleMapsLoader.VERSION = "weekly";
	  GoogleMapsLoader.KEY = "AIzaSyBFh-jqQ8z_YHhtAARbG0nVIZCGyGa_jN4";
	  GoogleMapsLoader.load(googleParam => {
	    google = googleParam;
	    map = new google.maps.Map(mapContainer, {
	      center: { lat: 0, lng: 0 },
	      zoom: 8
	    });
	    mapSize = calculateWidth(addressTableRow);
	  });
	});

	let map = null;
	let google = null;
	let mapSize = 0;
	let addressMarkers = [];
	let address = "";
	let addressLoading = false;
	let searchedAddresses = [];
	let nameSortAscending = null;

	let addressTableRow = null;
	let mapContainer = null;

	let sortedByNameSearchedAddresses = [];

	$: {
	  if (
	    !searchedAddresses ||
	    searchedAddresses.length === 0 ||
	    nameSortAscending == null
	  ) {
	    sortedByNameSearchedAddresses = searchedAddresses;
	  }
	}

	$: {
	  if (
	    searchedAddresses &&
	    searchedAddresses.length > 0 &&
	    nameSortAscending !== null
	  ) {
	    const sorted = [...searchedAddresses].sort((a, b) =>
	      a.properties.name.localeCompare(b.properties.name)
	    );

	    sortedByNameSearchedAddresses = nameSortAscending
	      ? sorted
	      : sorted.reverse();
	  }
	}

	async function getAddresses(newAddress) {
	  try {
	    newAddress = newAddress.trim();
	    if (addressMarkers.length > 0) {
	      for (const marker of addressMarkers) {
	        marker.setMap(null);
	      }
	      addressMarkers = [];
	    }

	    if (!address) {
	      searchedAddresses = [];
	      return;
	    }

	    addressLoading = true;

	    const response = await axios.get(
	      "https://api-adresse.data.gouv.fr/search",
	      {
	        params: {
	          q: address
	        }
	      }
	    );

	    const searchedAddressesResponse = response.data.features;

	    const bounds = new google.maps.LatLngBounds();
	    const addressMarkersNew = [];
	    searchedAddressesResponse.forEach(feature => {
	      const position = {
	        lat: feature.geometry.coordinates[1],
	        lng: feature.geometry.coordinates[0]
	      };

	      const marker = new google.maps.Marker({
	        position,
	        map: map,
	        title: feature.properties.name
	      });

	      bounds.extend(marker.getPosition());

	      addressMarkersNew.push(marker);
	    });

	    map.fitBounds(bounds);

	    searchedAddresses = searchedAddressesResponse;
	    addressMarkers = addressMarkersNew;
	  } catch (error) {
	    console.error(error);
	  } finally {
	    addressLoading = false;
	  }
	}

	function resize(event) {
	  mapSize = calculateWidth(addressTableRow);
	}
	function setNameSortAscending(params) {
	  nameSortAscending = sortDirectionMap[nameSortAscending];
	}
</script>

<style>
  /* Sticky footer styles */
  html {
    position: relative;
    min-height: 100%;
  }
  body {
    /* Margin bottom by footer height */
    margin-bottom: 60px;
  }
  .navbar {
    background-color: rgb(63, 81, 181);
  }
  .footer {
    position: fixed;
    bottom: 0;
    width: 100%;
    /* Set the fixed height of the footer here */
    height: 40px;
    line-height: 40px; /* Vertically center the text there */
    background-color: rgb(63, 81, 181);
  }
</style>
