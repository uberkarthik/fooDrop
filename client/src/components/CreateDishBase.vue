<template>
  <div class="create-dish-base">
    <header-base/>
    <v-layout>
      <v-flex xs12 sm6 offset-sm3>
        <v-card class='create-dish-card'>
          <v-form ref='form' v-model='valid' lazy-validation>
            <v-img v-if="newDishUrl != ''"
              :src='newDishUrl'
              aspect-ratio='2.75'
            ></v-img>
            <v-text-field label='Image URL' v-model='newDishUrl'></v-text-field>

            <v-text-field label='Name' v-model='newDishName' :rules='nameRules' class='dish-name'></v-text-field>
            <v-textarea label='Description' v-model='newDishDescription' rows='3' auto-grow></v-textarea>
            <v-layout row>
              <v-flex xs6>
                <v-text-field label='Latitude' v-model='newDishLocationLat' disabled></v-text-field>
              </v-flex>
              <v-flex xs6>
                <v-text-field label='Longitude' v-model='newDishLocationLong' disabled></v-text-field>
              </v-flex>
            </v-layout>

            <v-layout>
              <div id='map'></div>
            </v-layout>

            <v-combobox
              v-model='newDishDietaryRestrictions'
              :rules='dietaryRestrictionsRules'
              :items='dietaryRestrictionsList'
              label='Dietary Restrictions'
              chips
              clearable
              solo
              multiple
              dense
            >
              <template slot='selection' slot-scope='data'>
                <v-chip
                  :selected='data.selected'
                  close
                  @input='removeDietaryRestriction(data.item)'
                >
                  <strong>{{ data.item }}</strong>&nbsp;
                </v-chip>
              </template>
            </v-combobox>
            
            <v-combobox
              v-model='newDishIngredients'
              :items='[]'
              label='Ingredients'
              chips
              clearable
              solo
              multiple
            >
              <template slot='selection' slot-scope='data'>
                <v-chip
                  :selected='data.selected'
                  close
                  @input='removeIngredient(data.item)'
                >
                  <strong>{{ data.item }}</strong>&nbsp;
                </v-chip>
              </template>
            </v-combobox>

            <v-text-field label='Price' prefix='$' :rules='priceRules' v-model.number='newDishPrice'></v-text-field>
            <v-text-field label='Quantity' :rules='quantityRules' v-model.number='newDishQuantity'></v-text-field>

            <v-card-actions>
              <v-btn flat color='red' class='clearbtn' @click='clearForm'>x Clear</v-btn>
              <v-btn flat color='green' class='createbtn' :disabled='!valid' @click='createDish'>- Post</v-btn>
            </v-card-actions>
          </v-form>
        </v-card>
      </v-flex>
    </v-layout>
  </div>
</template>

<style>
  #map{
    height:400px;
    width:100%
  }
</style>

<script>
import DishesService from '@/services/DishesService'
import HeaderBase from '@/components/HeaderBase'
/**
 * @class CreateDishBase
 * @desc Component for creating dishes. Allows a user to enter dish details, select dish location, and create the dish on the backend server for sale.
 * @vue-data {GoogleMap} vueGMap - Google map display on the page for sale location selection
 * @vue-data {GoogleMapMarker} marker - Marker displaying the current dish sale position on the map
 * @vue-data {Boolean} valid - Validate the dish creation details
 * @vue-data {String} newDishName - Name of the new dish
 * @vue-data {String} newDishDescription - Description of the new dish
 * @vue-data {String} newDishUrl - Image URL of the new dish
 * @vue-data {Number} newDishLocationLat - Location (latitude) of the new dish
 * @vue-data {Number} newDishLocationLong - Location (longitude) of the new dish
 * @vue-data {Array.<String>} newDishIngredients - Ingredients of the new dish
 * @vue-data {Array.<String>} newDishDietaryRestrictions - Dietary restrictions of the new dish
 * @vue-data {Number} newDishPrice - Price of the new dish
 * @vue-data {Number} newDishQuantity - Quantity of the new dish available for sale
 */
export default {
  name: 'create-dish-base',
  components: {
    HeaderBase
  },
  data () {
    return {
      vueGMap: null,
      marker: null,
      position: null,

      // validation
      valid: true,
      nameRules: [(v) => { return !!v || 'Name is required' }],
      dietaryRestrictionsList: ['gluten free', 'vegan', 'vegetarian', 'lactose free', 'nut free'],
      dietaryRestrictionsRules: [(v) => { return (v && v.every((d) => { return this.dietaryRestrictionsList.includes(d); })) || 'Dietary Restrictions must be from dropdown!' }],
      priceRules: [
        (v) => { return !!v || 'Price is required'}, 
        (v) => { return (v && v > 0 && v <= 100) || 'Price must be between $0 and $100' }, 
        (v) => { return (v && (100*v - Math.floor(100*v) === 0)) || 'Price cannot have more than two decimal places' }
      ],
      quantityRules: [(v) => { return !!v || 'Quantity is required'}, (v) => { return (v && v > 0) || 'Quantity cannot be negative' }],

      // new dish details
      newDishName: '',
      newDishDescription: '',
      newDishUrl: '',
      newDishLocationLat: 0.0,
      newDishLocationLong: 0.0,
      newDishIngredients: [],
      newDishDietaryRestrictions: [],
      newDishPrice: 0,
      newDishQuantity: 0
    }
  },
  methods: {
    /**
     * Sends a request to create the dish on the backend database
     */
    async createDish () {
      if (this.$refs.form.validate()) {
        if (this.$store.state.paypalId && this.$store.state.paypalId !== '' && this.$store.state.paypalId !== '-1') {
          let response = await DishesService.newDish({
            name: this.newDishName,
            description: this.newDishDescription,
            imageUrl: this.newDishUrl,
            dietaryRestrictions: this.newDishDietaryRestrictions,
            ingredients: this.newDishIngredients,
            location: {lat: this.newDishLocationLat, lon: this.newDishLocationLong},
            price: this.newDishPrice,
            sellerId: this.$store.state.userId,
            sellerPaypalId: this.$store.state.paypalId,
            quantity: this.newDishQuantity
          })
          if (response.data.success) {
            alert('Success!')
            window.location.href = 'http://localhost:8080/'
          } else {
            alert('Error: ' + response.data.errorMessage)
          }
        } else {
          alert('You must enter your paypal email in your user profile to buy this dish')
        }
      }
    },
    /**
     * Clear the form data entered for the new dish
     */
    clearForm () {
      this.newDishName = ''
      this.newDishDescription = ''
      this.newDishUrl = ''
      this.newDishIngredients = []
      this.newDishDietaryRestrictions = []
      this.newDishPrice = null
      this.newDishQuantity = null
    },
    /**
     * Removes a dietary restriction from the new dish dietary restrictions list
     */
    removeDietaryRestriction (item) {
      this.newDishDietaryRestrictions.splice(this.newDishDietaryRestrictions.indexOf(item), 1)
      this.newDishDietaryRestrictions = [...this.newDishDietaryRestrictions]
    },
    /**
     * Removes an ingredient from the new dish ingredients list
     */
    removeIngredient (item) {
      this.newDishIngredients.splice(this.newDishIngredients.indexOf(item), 1)
      this.newDishIngredients = [...this.newDishIngredients]
    },
    /**
     * Place a marker on the Google map display
     */
    placeMarker(latlng, map) {
      console.log("Place Marker")
      if(this.marker != null) {
        this.marker.setMap(null)
      }

      this.marker = new google.maps.Marker({
        position: latlng,
        map: map
      })
      this.newDishLocationLat = latlng.lat
      this.newDishLocationLong = latlng.lng
      map.panTo(latlng)
    },
    /**
     * Initialize the Google map on the page for location selection
     */
    initGoogleMaps() {
      const localOptions = {
        zoom: 17,
        center: {lat: this.newDishLocationLat, lng: this.newDishLocationLong}
      }
      console.log("InitializeGoogle Maps")
      this.vueGMap = new google.maps.Map(document.getElementById('map'), localOptions)
      this.placeMarker({lat: this.newDishLocationLat, lng: this.newDishLocationLong}, this.vueGMap)
      this.vueGMap.addListener('click', (e) => {
        this.position = {lat: e.latLng.lat(), lng: e.latLng.lng()}
        this.placeMarker(this.position, this.vueGMap)
      })
    }
  },
  /**
   * Get the current location and inialize the google map
   */
  mounted () {
    if (navigator.geolocation) {
       var self = this;
       navigator.geolocation.getCurrentPosition(function (position) {
        self.newDishLocationLat = position.coords.latitude
        self.newDishLocationLong = position.coords.longitude
        self.initGoogleMaps()
      })
    }
  }
}
</script>

<style scoped>
.create-dish-card {
  margin-left: 1em;
  margin-right: 1em;
}

.dish-name {
  font-size: x-large;
}
</style>
