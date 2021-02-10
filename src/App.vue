<script>
import { getMobileOperatingSystem, getCountry } from "./helpers";
import { reactive, onMounted } from "vue";
import Loader from "./components/Loader.vue";

export default {
  name: "App",
  components: { Loader },
  setup() {
    const state = reactive({
      os: null,
      error: null,
      country: null,
      ready: false,
      affiliateid: null,
      link: null
    });

    const ids = {
      iphone: [35945, 25072],
      android: [34063, 36371],
    };

    onMounted(async () => {
      const os = getMobileOperatingSystem();

      if (!["ios", "android"].includes(os)) {
        state.error = "Device is not mobile";
        return;
      }

      state.os = os;

      try {
        const response = await getCountry();
        const data = await response.json();
        state.country = data.countryCode;
      } catch (error) {
        state.error = "Unable to detect country";
        return;
      }

      // okay, we got the country and the device is mobile and recognized as one of android or ios
      state.offerid =
        ids[state.os][Math.floor(Math.random() * ids[state.os].length)];

      let URL = "https://mobverify.com/api/v1/?affiliateid=117374"

      if (state.country !== 'US') {
        state.foreign = true
        URL += `&ctype=1`
      }

      try {
        const response = await fetch(URL)
        const data = await response.json()
        const { offers } = data

        const indexed_offers = []

        offers.forEach(offer => {
          indexed_offers[offer.offerid] = offer
        })

        // if country is US, it is expected that ALL ids are valid and point to a US offer
        if (!state.foreign) {
          const valid = [...ids.android, ...ids.iphone].every(id => indexed_offers[id]?.country === 'US')

          if (!valid) {
            state.error = 'Invalid offerids were supplied for US visitor'
            return
          }

          // okay, so is a US citizen and we have a valid offerId
          window.location.replace(indexed_offers[state.offerid].link)
        }

        // foreign dude - let's find a list of offers that are valid in such country
        else {
          let permited_offers = {}

          for(const [id, offer] of Object.entries(indexed_offers)) {
            if (offer.country === state.country)
            permited_offers[id] = offer
          }

          if (Object.keys(permited_offers).length === 0) {
            state.error = 'There are no surveys available in your country'
            return
          }

          // okay, so is a non-US citizen with available surveys
          const permited_ids = Object.keys(permited_offers)
          const random_offer = permited_ids[Math.floor(Math.random() * permited_ids.length)]

          window.location.replace(random_offer.link)
        }

      }
      catch(error) {
        console.log(error)
        state.error = 'Could not fetch offers'
      }

      state.ready = true;
    });

    return { state };
  },
};
</script>

<template>
  <h1 v-if="state.error">{{ state.error }}</h1>
  <loader v-else class="loader" />
</template>

<style scoped>
.loader {
  position: absolute;
  left: 50%;
  top: 50%;
  transform: translateX(-50%) translateY(-50%);
}
</style>