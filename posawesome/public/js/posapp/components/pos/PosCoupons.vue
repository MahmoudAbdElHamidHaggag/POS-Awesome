<template>
  <div class="pa-4">
    <v-card class="mx-auto elevation-1 rounded-lg" style="max-height: 85vh">
      <!-- Header Section -->
      <v-card-title class="bg-grey-lighten-3 pa-4">
        <v-row no-gutters align="center">
          <v-col cols="12" sm="4">
            <div class="d-flex align-center">
              <v-icon class="mr-3" color="grey-darken-2" size="24">mdi-ticket-percent</v-icon>
              <span class="text-h6 text-grey-darken-3">{{ __('Coupons') }}</span>
            </div>
          </v-col>
          <v-col cols="12" sm="8">
            <v-row no-gutters align="center">
              <v-col cols="8" sm="9">
                <v-text-field 
                  density="compact" 
                  variant="outlined" 
                  flat
                  :label="__('Enter Coupon Code')"
                  bg-color="white" 
                  hide-details 
                  v-model="new_coupon" 
                  class="mr-2"
                  prepend-inner-icon="mdi-barcode-scan"
                  @keyup.enter="add_coupon(new_coupon)"
                  clearable>
                </v-text-field>
              </v-col>
              <v-col cols="4" sm="3">
                <v-btn 
                  block
                  color="primary" 
                  variant="flat"
                  @click="add_coupon(new_coupon)"
                  :disabled="!new_coupon">
                  <v-icon size="small">mdi-plus</v-icon>
                  {{ __('Add') }}
                </v-btn>
              </v-col>
            </v-row>
          </v-col>
        </v-row>
      </v-card-title>

      <v-divider></v-divider>

      <!-- Statistics Bar -->
      <v-card-text class="pa-0">
        <div class="pa-3 bg-grey-lighten-4">
          <v-row no-gutters align="center">
            <v-col cols="6">
              <div class="d-flex align-center">
                <v-icon class="mr-2" size="small" color="grey-darken-1">mdi-ticket-outline</v-icon>
                <span class="text-body-2 text-grey-darken-2">{{ __('Total Coupons') }}: <strong>{{ couponsCount }}</strong></span>
              </div>
            </v-col>
            <v-col cols="6">
              <div class="d-flex align-center justify-end">
                <v-icon class="mr-2" size="small" color="teal-darken-1">mdi-check-circle</v-icon>
                <span class="text-body-2 text-grey-darken-2">{{ __('Applied') }}: <strong class="text-teal-darken-1">{{ appliedCouponsCount }}</strong></span>
              </div>
            </v-col>
          </v-row>
        </div>
      </v-card-text>

      <!-- Coupons Table -->
      <v-card-text class="pa-0" style="height: calc(85vh - 220px); overflow-y: auto;">
        <v-container fluid class="pa-0">
          <v-row v-if="posa_coupons.length === 0" class="ma-0">
            <v-col cols="12" class="text-center py-12">
              <v-icon size="64" color="grey-lighten-1">mdi-ticket-percent-outline</v-icon>
              <p class="text-h6 text-grey-darken-1 mt-4">{{ __('No coupons added yet') }}</p>
              <p class="text-body-2 text-grey">{{ __('Enter a coupon code above to get started') }}</p>
            </v-col>
          </v-row>

          <v-row v-else class="ma-0">
            <v-col cols="12" class="pa-3">
              <v-card 
                v-for="(coupon, index) in posa_coupons" 
                :key="coupon.coupon"
                class="mb-2 coupon-card"
                :class="{ 'applied-coupon': coupon.applied }"
                flat
                variant="outlined">
                <v-card-text class="pa-3">
                  <v-row no-gutters align="center">
                    <v-col cols="12" sm="6">
                      <div class="d-flex align-center">
                        <v-icon 
                          :color="coupon.applied ? 'teal-darken-1' : 'grey'" 
                          class="mr-3"
                          size="20">
                          {{ coupon.applied ? 'mdi-check-circle' : 'mdi-ticket-percent' }}
                        </v-icon>
                        <div>
                          <div class="text-body-2 font-weight-medium">{{ coupon.coupon_code }}</div>
                          <div class="text-caption text-grey-darken-1">{{ coupon.pos_offer }}</div>
                        </div>
                      </div>
                    </v-col>
                    <v-col cols="6" sm="3">
                      <v-chip 
                        :color="coupon.type === 'Promotional' ? 'deep-purple-lighten-4' : 'orange-lighten-4'" 
                        :text-color="coupon.type === 'Promotional' ? 'deep-purple-darken-2' : 'orange-darken-2'"
                        variant="flat"
                        size="small"
                        class="font-weight-regular">
                        {{ coupon.type }}
                      </v-chip>
                    </v-col>
                    <v-col cols="6" sm="3" class="text-right">
                      <span 
                        v-if="coupon.applied"
                        class="text-caption text-teal-darken-1 font-weight-medium">
                        <v-icon size="x-small" class="mr-1">mdi-check</v-icon>
                        {{ __('Applied') }}
                      </span>
                      <span 
                        v-else
                        class="text-caption text-grey">
                        {{ __('Not Applied') }}
                      </span>
                    </v-col>
                  </v-row>
                </v-card-text>
              </v-card>
            </v-col>
          </v-row>
        </v-container>
      </v-card-text>

      <v-divider></v-divider>

      <!-- Footer Actions -->
      <v-card-actions class="pa-4 bg-grey-lighten-5">
        <v-btn 
          block 
          size="large" 
          color="grey-darken-2" 
          variant="flat"
          @click="back_to_invoice"
          prepend-icon="mdi-arrow-left">
          {{ __('Back to Invoice') }}
        </v-btn>
      </v-card-actions>
    </v-card>
  </div>
</template>

<script>
export default {
  data: () => ({
    loading: false,
    pos_profile: '',
    customer: '',
    posa_coupons: [],
    new_coupon: null,
    itemsPerPage: 1000,
    singleExpand: true,
  }),

  computed: {
    couponsCount() {
      return this.posa_coupons.length;
    },
    appliedCouponsCount() {
      return this.posa_coupons.filter((el) => !!el.applied).length;
    },
  },

  methods: {
    back_to_invoice() {
      this.eventBus.emit('show_coupons', 'false');
    },
    add_coupon(new_coupon) {
      if (!this.customer || !new_coupon) {
        this.eventBus.emit('show_message', {
          title: __('Select a customer to use coupon'),
          color: 'error',
        });
        return;
      };
      const exist = this.posa_coupons.find(
        (el) => el.coupon_code == new_coupon
      );
      if (exist) {
        this.eventBus.emit('show_message', {
          title: __('This coupon already used !'),
          color: 'error',
        });
        return;
      }
      const vm = this;
      frappe.call({
        method: 'posawesome.posawesome.api.posapp.get_pos_coupon',
        args: {
          coupon: new_coupon,
          customer: vm.customer,
          company: vm.pos_profile.company,
        },
        callback: function (r) {
          if (r.message) {
            const res = r.message;
            if (res.msg != 'Apply' || !res.coupon) {
              vm.eventBus.emit('show_message', {
                text: res.msg,
                color: 'error',
              });
            } else {
              vm.new_coupon = null;
              const coupon = res.coupon;
              vm.posa_coupons.push({
                coupon: coupon.name,
                coupon_code: coupon.coupon_code,
                type: coupon.coupon_type,
                applied: 0,
                pos_offer: coupon.pos_offer,
                customer: coupon.customer || vm.customer,
              });
            }
          }
        },
      });
    },
    setActiveGiftCoupons() {
      if (!this.customer) return;
      const vm = this;
      frappe.call({
        method: 'posawesome.posawesome.api.posapp.get_active_gift_coupons',
        args: {
          customer: vm.customer,
          company: vm.pos_profile.company,
        },
        callback: function (r) {
          if (r.message) {
            const coupons = r.message;
            coupons.forEach((coupon_code) => {
              vm.add_coupon(coupon_code);
            });
          }
        },
      });
    },

    updatePosCoupons(offers) {
      this.posa_coupons.forEach((coupon) => {
        const offer = offers.find(
          (el) => el.offer_applied && el.coupon == coupon.coupon
        );
        if (offer) {
          coupon.applied = 1;
        } else {
          coupon.applied = 0;
        }
      });
    },

    removeCoupon(reomove_list) {
      this.posa_coupons = this.posa_coupons.filter(
        (coupon) => !reomove_list.includes(coupon.coupon)
      );
    },
    updateInvoice() {
      this.eventBus.emit('update_invoice_coupons', this.posa_coupons);
    },
    updateCounters() {
      this.eventBus.emit('update_coupons_counters', {
        couponsCount: this.couponsCount,
        appliedCouponsCount: this.appliedCouponsCount,
      });
    },
  },

  watch: {
    posa_coupons: {
      deep: true,
      handler() {
        this.updateInvoice();
        this.updateCounters();
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
        const to_remove = [];
        this.posa_coupons.forEach((el) => {
          if (el.type == 'Promotional') {
            el.customer = customer;
          } else {
            to_remove.push(el.coupon);
          }
        });
        this.customer = customer;
        if (to_remove.length) {
          this.removeCoupon(to_remove);
        }
      }
      this.setActiveGiftCoupons();
    });
    this.eventBus.on('update_pos_coupons', (data) => {
      this.updatePosCoupons(data);
    });
    this.eventBus.on('set_pos_coupons', (data) => {
      this.posa_coupons = data;
    });
  },
};
</script>

<style scoped>
.coupon-card {
  transition: all 0.2s ease;
  border-left: 3px solid transparent;
}

.coupon-card:hover {
  transform: translateX(-2px);
  background-color: #FAFAFA;
}

.applied-coupon {
  background-color: #F5F5F5;
  border-left-color: #00897B;
}

.v-chip {
  font-size: 0.75rem;
}

/* تحسينات الرسوم المتحركة */
.v-card {
  transition: all 0.2s ease;
}

.v-btn {
  text-transform: none;
  font-weight: 400;
}

/* تحسين شكل حقل الإدخال */
.v-text-field :deep(.v-field__input) {
  font-size: 0.875rem;
}

/* تحسين التمرير */
::-webkit-scrollbar {
  width: 6px;
}

::-webkit-scrollbar-track {
  background: #f5f5f5;
}

::-webkit-scrollbar-thumb {
  background: #e0e0e0;
  border-radius: 3px;
}

::-webkit-scrollbar-thumb:hover {
  background: #d0d0d0;
}

/* دعم RTL */
[dir="rtl"] .mr-3 {
  margin-right: 0 !important;
  margin-left: 0.75rem !important;
}

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

[dir="rtl"] .coupon-card {
  border-left: none !important;
  border-right: 3px solid transparent;
}

[dir="rtl"] .applied-coupon {
  border-right-color: #00897B;
}

[dir="rtl"] .coupon-card:hover {
  transform: translateX(2px);
}
</style>