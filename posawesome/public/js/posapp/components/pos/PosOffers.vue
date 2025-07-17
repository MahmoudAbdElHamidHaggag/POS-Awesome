<template>
  <div>
    <v-card class="mx-auto elevation-0" style="max-height: 88vh; height: 88vh">
      <!-- Header -->
      <v-card-title class="pa-4 bg-gradient-primary">
        <div class="d-flex align-center">
          <v-icon class="mr-2" color="white" size="28">mdi-tag-multiple</v-icon>
          <span class="text-h5 text-white">{{ __('Special Offers') }}</span>
        </div>
      </v-card-title>

      <!-- Statistics Bar -->
      <v-alert 
        class="ma-0 rounded-0" 
        color="info" 
        variant="tonal"
        density="compact">
        <v-row no-gutters align="center">
          <v-col cols="6">
            <div class="d-flex align-center">
              <v-icon class="mr-2" size="small">mdi-tag-outline</v-icon>
              <span class="text-body-1">{{ __('Available Offers') }}: <strong>{{ offersCount }}</strong></span>
            </div>
          </v-col>
          <v-col cols="6">
            <div class="d-flex align-center justify-end">
              <v-icon class="mr-2" size="small" color="success">mdi-check-circle</v-icon>
              <span class="text-body-1">{{ __('Applied') }}: <strong class="text-success">{{ appliedOffersCount }}</strong></span>
            </div>
          </v-col>
        </v-row>
      </v-alert>

      <!-- Offers List -->
      <v-card-text class="pa-0 bg-grey-lighten-5" style="height: calc(88vh - 180px); overflow-y: auto;">
        <!-- Empty State -->
        <div v-if="pos_offers.length === 0" class="text-center py-16">
          <v-icon size="80" color="grey-lighten-1">mdi-tag-off-outline</v-icon>
          <p class="text-h5 text-grey-darken-1 mt-4 mb-2">{{ __('No offers available') }}</p>
          <p class="text-body-1 text-grey">{{ __('Check back later for special deals') }}</p>
        </div>

        <!-- Offers Cards -->
        <v-container v-else fluid class="pa-4">
          <v-row>
            <v-col 
              v-for="(offer, index) in pos_offers" 
              :key="offer.row_id"
              cols="12"
              md="6"
              lg="4"
              class="py-2">
              <v-card 
                class="offer-card h-100" 
                :class="{ 'offer-applied': offer.offer_applied }"
                variant="outlined"
                @click="expanded = expanded.includes(offer.row_id) ? expanded.filter(id => id !== offer.row_id) : [...expanded, offer.row_id]">
                
                <v-card-text class="pa-4">
                  <!-- Header Row -->
                  <v-row no-gutters align="center" class="mb-3">
                    <v-col>
                      <div class="d-flex align-center">
                        <v-icon 
                          :color="getOfferIcon(offer.offer).color" 
                          size="24"
                          class="mr-2">
                          {{ getOfferIcon(offer.offer).icon }}
                        </v-icon>
                        <span class="text-subtitle-1 font-weight-bold">{{ offer.name }}</span>
                      </div>
                    </v-col>
                    <v-col cols="auto">
                      <v-checkbox
                        v-model="offer.offer_applied"
                        @click.stop="toggleOfferApplied(offer)"
                        :disabled="isOfferDisabled(offer)"
                        color="success"
                        hide-details>
                      </v-checkbox>
                    </v-col>
                  </v-row>

                  <!-- Info Section -->
                  <div class="mb-3">
                    <div class="text-body-2 text-grey-darken-2 mb-1">
                      <v-icon size="x-small" class="mr-1">mdi-label</v-icon>
                      <span class="font-weight-medium">{{ __('Type') }}:</span> {{ __(offer.offer) }}
                    </div>
                    
                    <div class="text-body-2 text-grey-darken-2 mb-1">
                      <v-icon size="x-small" class="mr-1">mdi-target</v-icon>
                      <span class="font-weight-medium">{{ __('Applies to') }}:</span> {{ offer.apply_on }}
                    </div>

                    <div v-if="offer.auto" class="mt-2">
                      <v-chip 
                        color="orange-lighten-4"
                        text-color="orange-darken-2"
                        size="small"
                        variant="flat"
                        prepend-icon="mdi-lightning-bolt">
                        {{ __('Auto Apply') }}
                      </v-chip>
                    </div>
                  </div>

                  <!-- Expand/Collapse Button -->
                  <v-btn
                    variant="text"
                    size="small"
                    color="primary"
                    block
                    @click.stop="expanded = expanded.includes(offer.row_id) ? expanded.filter(id => id !== offer.row_id) : [...expanded, offer.row_id]">
                    <v-icon size="small" class="mr-1">
                      {{ expanded.includes(offer.row_id) ? 'mdi-chevron-up' : 'mdi-chevron-down' }}
                    </v-icon>
                    {{ expanded.includes(offer.row_id) ? __('Show Less') : __('Show More') }}
                  </v-btn>

                  <!-- Expanded Content -->
                  <v-expand-transition>
                    <div v-if="expanded.includes(offer.row_id)">
                      <v-divider class="my-3"></v-divider>
                      
                      <div v-if="offer.description" class="text-body-2 text-grey-darken-1 mb-3 pa-2 bg-grey-lighten-4 rounded">
                        <div v-html="handleNewLine(offer.description)"></div>
                      </div>

                      <div v-if="offer.offer == 'Give Product'" class="mt-3">
                        <v-autocomplete
                          v-model="offer.give_item"
                          :items="get_give_items(offer)"
                          item-title="item_code"
                          variant="outlined"
                          density="compact"
                          color="primary"
                          :label="__('Select free item')"
                          :disabled="offer.apply_type != 'Item Group' || offer.replace_item || offer.replace_cheapest_item"
                          hide-details
                          @click.stop>
                          <template v-slot:prepend-inner>
                            <v-icon size="small">mdi-gift</v-icon>
                          </template>
                        </v-autocomplete>
                      </div>

                      <v-alert 
                        v-if="offer.offer_applied && offer.items && offer.items.length > 0"
                        color="success"
                        variant="tonal"
                        density="compact"
                        class="mt-3">
                        <v-icon size="small">mdi-check-all</v-icon>
                        {{ __('Applied to') }} {{ offer.items.length }} {{ offer.items.length === 1 ? __('item') : __('items') }}
                      </v-alert>
                    </div>
                  </v-expand-transition>
                </v-card-text>
              </v-card>
            </v-col>
          </v-row>
        </v-container>
      </v-card-text>
    </v-card>

    <!-- Bottom Action Bar -->
    <v-card flat class="mt-2" style="height: 10vh">
      <v-container fluid class="pa-2 h-100">
        <v-row align="center" class="h-100">
          <v-col cols="6">
            <v-btn 
              block 
              size="large" 
              color="grey-darken-2" 
              variant="flat"
              @click="back_to_invoice"
              prepend-icon="mdi-arrow-left">
              {{ __('Back') }}
            </v-btn>
          </v-col>
          <v-col cols="6">
            <v-btn 
              block 
              size="large" 
              color="success" 
              variant="flat"
              @click="applyOffers"
              :disabled="appliedOffersCount === 0"
              prepend-icon="mdi-check">
              {{ __('Apply') }} ({{ appliedOffersCount }})
            </v-btn>
          </v-col>
        </v-row>
      </v-container>
    </v-card>
  </div>
</template>

<script>
import format from '../../format';

export default {
  mixins: [format],
  data: () => ({
    loading: false,
    pos_profile: '',
    pos_offers: [],
    allItems: [],
    discount_percentage_offer_name: null,
    expanded: [],
  }),

  computed: {
    offersCount() {
      return this.pos_offers.length;
    },
    appliedOffersCount() {
      return this.pos_offers.filter((el) => !!el.offer_applied).length;
    },
  },

  methods: {
    back_to_invoice() {
      this.eventBus.emit('show_offers', 'false');
    },
    
    applyOffers() {
      this.eventBus.emit('show_message', {
        title: __('Offers applied successfully'),
        color: 'success',
      });
      this.back_to_invoice();
    },
    
    toggleOfferApplied(offer) {
      if (!this.isOfferDisabled(offer)) {
        offer.offer_applied = !offer.offer_applied;
        this.forceUpdateItem();
      }
    },
    
    isOfferDisabled(offer) {
      return (offer.offer == 'Give Product' &&
        !offer.give_item &&
        (!offer.replace_cheapest_item || !offer.replace_item)) ||
        (offer.offer == 'Grand Total' &&
          this.discount_percentage_offer_name &&
          this.discount_percentage_offer_name != offer.name);
    },
    
    getOfferIcon(offerType) {
      const icons = {
        'Discount Percentage': { icon: 'mdi-percent', color: 'blue' },
        'Discount Amount': { icon: 'mdi-cash-minus', color: 'green' },
        'Give Product': { icon: 'mdi-gift', color: 'purple' },
        'Grand Total': { icon: 'mdi-calculator', color: 'orange' },
        'Price': { icon: 'mdi-tag', color: 'red' }
      };
      return icons[offerType] || { icon: 'mdi-tag', color: 'grey' };
    },
    
    forceUpdateItem() {
      let list_offers = [];
      list_offers = [...this.pos_offers];
      this.pos_offers = list_offers;
    },
    
    makeid(length) {
      let result = '';
      const characters = 'abcdefghijklmnopqrstuvwxyz0123456789';
      const charactersLength = characters.length;
      for (var i = 0; i < length; i++) {
        result += characters.charAt(
          Math.floor(Math.random() * charactersLength)
        );
      }
      return result;
    },
    
    updatePosOffers(offers) {
      const toRemove = [];
      this.pos_offers.forEach((pos_offer) => {
        const offer = offers.find((offer) => offer.name === pos_offer.name);
        if (!offer) {
          toRemove.push(pos_offer.row_id);
        }
      });
      this.removeOffers(toRemove);
      
      offers.forEach((offer) => {
        const pos_offer = this.pos_offers.find(
          (pos_offer) => offer.name === pos_offer.name
        );
        if (pos_offer) {
          pos_offer.items = offer.items;
          if (
            pos_offer.offer === 'Grand Total' &&
            !this.discount_percentage_offer_name
          ) {
            pos_offer.offer_applied = !!pos_offer.auto;
          }
          if (
            offer.apply_on == 'Item Group' &&
            offer.apply_type == 'Item Group' &&
            offer.replace_cheapest_item
          ) {
            pos_offer.give_item = offer.give_item;
            pos_offer.apply_item_code = offer.apply_item_code;
          }
        } else {
          const newOffer = { ...offer };
          if (!offer.row_id) {
            newOffer.row_id = this.makeid(20);
          }
          if (offer.apply_type == 'Item Code') {
            newOffer.give_item = offer.apply_item_code || 'Nothing';
          }
          if (offer.offer_applied) {
            newOffer.offer_applied == !!offer.offer_applied;
          } else {
            if (
              offer.apply_type == 'Item Group' &&
              offer.offer == 'Give Product' &&
              !offer.replace_cheapest_item &&
              !offer.replace_item
            ) {
              newOffer.offer_applied = false;
            } else if (
              offer.offer === 'Grand Total' &&
              this.discount_percentage_offer_name
            ) {
              newOffer.offer_applied = false;
            } else {
              newOffer.offer_applied = !!offer.auto;
            }
          }
          if (newOffer.offer == 'Give Product' && !newOffer.give_item) {
            newOffer.give_item = this.get_give_items(newOffer)[0]?.item_code;
          }
          this.pos_offers.push(newOffer);
          this.eventBus.emit('show_message', {
            title: __('New offer available'),
            color: 'success',
          });
        }
      });
    },
    
    removeOffers(offers_id_list) {
      this.pos_offers = this.pos_offers.filter(
        (offer) => !offers_id_list.includes(offer.row_id)
      );
    },
    
    handelOffers() {
      const applyedOffers = this.pos_offers.filter(
        (offer) => offer.offer_applied
      );
      this.eventBus.emit('update_invoice_offers', applyedOffers);
    },
    
    handleNewLine(str) {
      if (str) {
        return str.replace(/(?:\r\n|\r|\n)/g, '<br />');
      } else {
        return '';
      }
    },
    
    get_give_items(offer) {
      if (offer.apply_type == 'Item Code') {
        return [{ item_code: offer.apply_item_code }];
      } else if (offer.apply_type == 'Item Group') {
        const items = this.allItems;
        let filterd_items = [];
        const filterd_items_1 = items.filter(
          (item) => item.item_group == offer.apply_item_group
        );
        if (offer.less_then > 0) {
          filterd_items = filterd_items_1.filter(
            (item) => item.rate < offer.less_then
          );
        } else {
          filterd_items = filterd_items_1;
        }
        return filterd_items;
      } else {
        return [];
      }
    },
    
    updateCounters() {
      this.eventBus.emit('update_offers_counters', {
        offersCount: this.offersCount,
        appliedOffersCount: this.appliedOffersCount,
      });
    },
    
    updatePosCoupuns() {
      const applyedOffers = this.pos_offers.filter(
        (offer) => offer.offer_applied && offer.coupon_based
      );
      this.eventBus.emit('update_pos_coupons', applyedOffers);
    },
  },

  watch: {
    pos_offers: {
      deep: true,
      handler(pos_offers) {
        this.handelOffers();
        this.updateCounters();
        this.updatePosCoupuns();
      },
    },
  },

  created: function () {
    this.$nextTick(function () {
      this.eventBus.on('register_pos_profile', (data) => {
        this.pos_profile = data.pos_profile;
      });
    });
    
    this.eventBus.on('update_customer', (customer) => {
      if (this.customer != customer) {
        this.offers = [];
      }
    });
    
    this.eventBus.on('update_pos_offers', (data) => {
      this.updatePosOffers(data);
    });
    
    this.eventBus.on('update_discount_percentage_offer_name', (data) => {
      this.discount_percentage_offer_name = data.value;
    });
    
    this.eventBus.on('set_all_items', (data) => {
      this.allItems = data;
    });
  },
};
</script>

<style scoped>
.offer-card {
  transition: all 0.2s ease;
  cursor: pointer;
  height: 100%;
  display: flex;
  flex-direction: column;
}

.offer-card:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
}

.offer-applied {
  background-color: #E8F5E9;
  border-color: #4CAF50 !important;
  border-width: 2px !important;
}

.bg-gradient-primary {
  background: linear-gradient(135deg, #1976D2 0%, #1565C0 100%);
}

.h-100 {
  height: 100%;
}

/* تحسين التمرير */
::-webkit-scrollbar {
  width: 8px;
}

::-webkit-scrollbar-track {
  background: #f5f5f5;
}

::-webkit-scrollbar-thumb {
  background: #d5d5d5;
  border-radius: 4px;
}

::-webkit-scrollbar-thumb:hover {
  background: #c0c0c0;
}

/* دعم RTL */
[dir="rtl"] .mr-2 {
  margin-right: 0 !important;
  margin-left: 0.5rem !important;
}

[dir="rtl"] .mr-1 {
  margin-right: 0 !important;
  margin-left: 0.25rem !important;
}

[dir="rtl"] .text-right {
  text-align: left !important;
}

/* إزالة text-transform من الأزرار */
.v-btn {
  text-transform: none;
  letter-spacing: normal;
}

/* تحسين مظهر البطاقات على الشاشات الصغيرة */
@media (max-width: 960px) {
  .offer-card {
    margin-bottom: 8px;
  }
}
</style>