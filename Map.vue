<template>
  <div id="map-container">
    <div class="q-pa-xl absolute bg-white text-center" v-show="mapState == 'empty'" id="map-empty">
      <q-icon size="xl" name="fas fa-map"></q-icon>
      <br>
      <br>
      Mapa Vazio.
    </div>
    <div class="q-pa-xl absolute bg-white text-center" v-show="mapState == 'loading'" id="map-loading">
      <div>
        <q-spinner-oval size="lg" />
      </div>
      <div class="text-caption">
        Carregando...
      </div>
    </div>
    <div id="map" v-show="mapState == 'ready'"></div>
  </div>
</template>
<script>
export default {
  name: 'component-map',

  props: {
    AddressObj: Object,
    AddressStr: String
  },

  data() {
    return {
      mapState: 'empty',
      updMapDebounce: null,
    }
  },

  watch: {
    AddressObj: {
      handler(val) {
        if (!!val && !!val.ds_street && !!val.ds_number && !!val.ds_neighborhood && !!val.ds_city && !!val.do_uf) {
          this.updMapAddress(true);
        } else this.mapState = 'empty';
      },
      deep: true
    },

    AddressStr(val) {
      if (!this.AddressStr) this.mapState = 'empty';
      else this.updMapAddress(false);
    }
  },

  methods: {
    buildFullAddress(address) {
      if (!address || address.ds_street == null) return null;

      var fullAddress = address.ds_street +
        ", " + address.ds_number +
        (address.ds_complement ? ", " + address.ds_complement : '') +
        ", " + address.ds_neighborhood +
        ", " + address.ds_city +
        "/" + address.do_uf +
        " - CEP: " + address.ds_zipcode;

      return fullAddress;
    },

    async getGeoCode(address, callback) {
      var geocoder = new google.maps.Geocoder();

      await geocoder.geocode({
        'address': address
      }, function (results, status) {

        if (status == google.maps.GeocoderStatus.OK) {
          let latitude = results[0].geometry.location.lat();
          let longitude = results[0].geometry.location.lng();

          if (callback)
            callback(latitude, longitude);
        }
      });
    },

    updMapAddress(buildAddress) {
      buildAddress = buildAddress != null ? buildAddress : true;

      this.mapState = 'loading';

      clearTimeout(this.updMapDebounce);

      this.updMapDebounce = setTimeout(() => {
        var fullAddress = buildAddress ? this.buildFullAddress(this.AddressObj) : this.AddressStr;
        if (fullAddress == null) return;

        this.getGeoCode(fullAddress, (latitude, longitude) => {
          this.initMap(latitude, longitude);
        });
      }, 300);
    },

    // Initialize and add the map
    initMap(lat, lng) {
      // The location
      var location = {
        lat: lat,
        lng: lng
      };
      // The map, centered at location
      const map = new google.maps.Map(document.getElementById("map"), {
        zoom: 14,
        center: location,
      });

      this.map = map;

      // The marker, positioned at location
      const marker = new google.maps.Marker({
        position: location,
        map: map,
      });

      this.mapState = 'ready';
    },
  },
}
</script>
<style scoped>
#map-container {
  position: relative;
  height: 300px;
}

#map {
  height: 300px;
}

#map-container>.absolute {
  width: 100%;
  height: 300px;
}
</style>