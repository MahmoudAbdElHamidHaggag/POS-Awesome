<template>
  <div class="invoice-container" :dir="isRTL ? 'rtl' : 'ltr'">
    <v-dialog v-model="cancel_dialog" max-width="320">
      <v-card>
        <v-card-title class="text-h6 text-error">
          {{ __("Cancel Sale?") }}
        </v-card-title>
        <v-card-text class="text-body-2">
          {{ __("This will cancel and delete the current sale. To save it as Draft, click 'Save and Clear' instead.") }}
        </v-card-text>
        <v-card-actions>
          <v-spacer></v-spacer>
          <v-btn color="error" variant="text" @click="cancel_invoice">
            {{ __("Yes, Cancel") }}
          </v-btn>
          <v-btn color="grey" variant="text" @click="cancel_dialog = false">
            {{ __("Back") }}
          </v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>

    <v-card class="invoice-card" elevation="2">
      <div class="header-section">
        <v-row dense class="pa-2">
          <v-col :cols="pos_profile.posa_allow_sales_order ? 8 : 12">
            <Customer />
          </v-col>
          <v-col v-if="pos_profile.posa_allow_sales_order" cols="4">
            <v-select
              v-model="invoiceType"
              :items="invoiceTypes"
              :label="frappe._('Type')"
              variant="outlined"
              density="compact"
              hide-details
              :disabled="invoiceType == 'Return'"
              class="type-select"
            />
          </v-col>
        </v-row>

        <v-row v-if="pos_profile.posa_use_delivery_charges" dense class="pa-2 pt-0">
          <v-col cols="8">
            <v-autocomplete
              v-model="selected_delivery_charge"
              :items="delivery_charges"
              item-title="name"
              item-value="name"
              return-object
              :label="frappe._('Delivery Charges')"
              variant="outlined"
              density="compact"
              hide-details
              clearable
              :disabled="readonly"
              @update:model-value="update_delivery_charges()"
              class="delivery-select"
            >
              <template v-slot:item="{ props, item }">
                <v-list-item v-bind="props">
                  <v-list-item-title>{{ item.raw.name }}</v-list-item-title>
                  <v-list-item-subtitle>{{ __("Rate") }}: {{ item.raw.rate }}</v-list-item-subtitle>
                </v-list-item>
              </template>
            </v-autocomplete>
          </v-col>
          <v-col cols="4">
            <v-text-field
              :model-value="formatCurrency(delivery_charges_rate)"
              :label="frappe._('Rate')"
              variant="outlined"
              density="compact"
              hide-details
              disabled
              class="rate-field"
            />
          </v-col>
        </v-row>

        <v-row v-if="pos_profile.posa_allow_change_posting_date" dense class="pa-2 pt-0">
          <v-col cols="6">
            <v-menu v-model="invoice_posting_date" :close-on-content-click="false">
              <template v-slot:activator="{ props }">
                <v-text-field
                  v-model="posting_date"
                  :label="frappe._('Posting Date')"
                  variant="outlined"
                  density="compact"
                  hide-details
                  readonly
                  v-bind="props"
                  class="date-field"
                />
              </template>
              <v-date-picker
                v-model="posting_date"
                color="primary"
                :min="frappe.datetime.add_days(frappe.datetime.now_date(true), -7)"
                :max="frappe.datetime.add_days(frappe.datetime.now_date(true), 7)"
                @update:model-value="invoice_posting_date = false"
              />
            </v-menu>
          </v-col>
        </v-row>
      </div>

      <div class="items-section">
        <div v-if="items.length === 0" class="empty-cart">
          <div class="empty-cart-content">
            <div class="empty-cart-icon">
              <svg viewBox="0 0 24 24" fill="none" xmlns="http://www.w3.org/2000/svg">
                <path d="M1 1H5L7.68 14.39C7.77144 14.8504 8.02191 15.264 8.38755 15.5583C8.75318 15.8526 9.2107 16.009 9.68 16H19.4C19.8693 16.009 20.3268 15.8526 20.6925 15.5583C21.0581 15.264 21.3086 14.8504 21.4 14.39L23 6H6" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/>
                <circle cx="8" cy="21" r="1" stroke="currentColor" stroke-width="2"/>
                <circle cx="19" cy="21" r="1" stroke="currentColor" stroke-width="2"/>
                <path d="M12 3V12M9 9L12 12L15 9" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/>
              </svg>
            </div>
            <h3 class="empty-cart-title">{{ __("Cart is Empty") }}</h3>
            <p class="empty-cart-subtitle">{{ __("Start adding products to continue") }}</p>
          </div>
        </div>
        <div v-else class="items-list">
          <div v-for="(item, index) in items" :key="item.posa_row_id" class="item-card">
            <div class="item-content">
              <div class="item-header-line" @click="toggleItemDetails(item)">
                <div class="item-title">
                  <span class="item-name">{{ item.item_name }}</span>
                  <span class="separator">|</span>
                  <span class="item-code">{{ item.item_code }}</span>
                </div>
                <v-chip 
                  v-if="!!item.posa_is_offer || !!item.posa_is_replace" 
                  size="x-small"
                  color="success"
                  class="offer-chip"
                >
                  {{ __("Offer") }}
                </v-chip>
              </div>
              
              <div class="item-data-line">
                <div class="action-group">
                  <v-btn
                    icon
                    size="x-small"
                    variant="text"
                    color="error"
                    :disabled="!!item.posa_is_offer || !!item.posa_is_replace"
                    @click.stop="remove_item(item)"
                    class="delete-btn"
                  >
                    <v-icon>mdi-delete</v-icon>
                  </v-btn>
                </div>
                
                <div class="data-group total-group">
                  <span class="total-value">{{ formatCurrency(parseFloat(item.qty || 0) * parseFloat(item.rate || 0)) }}</span>
                </div>
                
                <div class="data-group price-group" @click.stop>
                  <input
                    type="text"
                    :value="formatPrice(item.rate)"
                    @input="handlePriceInputTemp(item, $event)"
                    @blur="handlePriceCommit(item)"
                    @keydown.enter="handlePriceCommit(item)"
                    @keydown.tab="handlePriceCommit(item)"
                    @click.stop
                    :disabled="!!item.posa_is_offer || !!item.posa_is_replace || !!item.posa_offer_applied || !pos_profile.posa_allow_user_to_edit_rate || !!invoice_doc.is_return"
                    class="price-input"
                    inputmode="decimal"
                    pattern="[0-9]*[.,]?[0-9]*"
                  />
                </div>
                
                <div class="data-group uom-group" @click.stop>
                  <v-select
                    v-model="item.uom"
                    :items="item.item_uoms || []"
                    item-title="uom"
                    item-value="uom"
                    variant="plain"
                    density="compact"
                    hide-details
                    single-line
                    :disabled="!!invoice_doc.is_return || !!item.posa_is_offer || !!item.posa_is_replace"
                    @update:model-value="calc_uom(item, $event)"
                    class="uom-select"
                  />
                </div>
                
                <div class="data-group qty-group" @click.stop>
                  <v-btn
                    icon
                    size="x-small"
                    variant="text"
                    density="compact"
                    :disabled="!!item.posa_is_offer || !!item.posa_is_replace || !!invoice_doc.is_return"
                    @click.stop="add_one(item)"
                    class="qty-btn"
                  >
                    <v-icon>mdi-plus</v-icon>
                  </v-btn>
                  <input
                    type="number"
                    v-model.number="item.qty"
                    @input="handleQtyChange(item)"
                    @click.stop
                    :disabled="!!invoice_doc.is_return || !!item.posa_is_offer || !!item.posa_is_replace"
                    class="qty-input"
                    min="0"
                    step="1"
                  />
                  <v-btn
                    icon
                    size="x-small"
                    variant="text"
                    density="compact"
                    :disabled="!!item.posa_is_offer || !!item.posa_is_replace || !!invoice_doc.is_return"
                    @click.stop="subtract_one(item)"
                    class="qty-btn"
                  >
                    <v-icon>mdi-minus</v-icon>
                  </v-btn>
                </div>
              </div>
            </div>

            <v-collapse-transition>
              <div v-if="isItemExpanded(item)" class="item-details">
                <v-row dense>
                  <v-col cols="4">
                    <div class="detail-field">
                      <label>{{ __("Quantity") }}</label>
                      <v-text-field
                        v-model.number="item.qty"
                        type="number"
                        variant="outlined"
                        density="compact"
                        hide-details
                        :disabled="!!invoice_doc.is_return || !!item.posa_is_offer || !!item.posa_is_replace"
                        @change="handleQtyChange(item)"
                        class="compact-input"
                      />
                    </div>
                  </v-col>
                  
                  <v-col cols="4">
                    <div class="detail-field">
                      <label>{{ __("UOM") }}</label>
                      <v-select
                        v-model="item.uom"
                        :items="item.item_uoms || []"
                        item-title="uom"
                        item-value="uom"
                        variant="outlined"
                        density="compact"
                        hide-details
                        :disabled="!!invoice_doc.is_return || !!item.posa_is_offer || !!item.posa_is_replace"
                        @update:model-value="calc_uom(item, $event)"
                        class="compact-input"
                      />
                    </div>
                  </v-col>
                  
                  <v-col cols="4">
                    <div class="detail-field">
                      <label>{{ __("Rate") }}</label>
                      <v-text-field
                        :model-value="formatPrice(item.rate)"
                        @input="handlePriceInputTemp(item, $event)"
                        @blur="handlePriceCommit(item)"
                        @keydown.enter="handlePriceCommit(item)"
                        @keydown.tab="handlePriceCommit(item)"
                        variant="outlined"
                        density="compact"
                        hide-details
                        :disabled="!!item.posa_is_offer || !!item.posa_is_replace || !!item.posa_offer_applied || !pos_profile.posa_allow_user_to_edit_rate || !!invoice_doc.is_return"
                        class="compact-input"
                      />
                    </div>
                  </v-col>
                </v-row>

                <v-row dense v-if="item.has_serial_no || item.has_batch_no || (pos_profile.posa_allow_sales_order && invoiceType == 'Order')">
                  <v-col cols="12" v-if="item.has_serial_no">
                    <v-autocomplete
                      v-model="item.serial_no_selected"
                      :items="item.serial_no_data"
                      item-title="serial_no"
                      :label="__('Serial Numbers')"
                      variant="outlined"
                      density="compact"
                      chips
                      multiple
                      hide-details
                      @update:model-value="set_serial_no(item)"
                      class="mt-2"
                    />
                  </v-col>
                  
                  <v-col cols="12" v-if="item.has_batch_no">
                    <v-autocomplete
                      v-model="item.batch_no"
                      :items="item.batch_no_data"
                      item-title="batch_no"
                      :label="__('Batch No')"
                      variant="outlined"
                      density="compact"
                      hide-details
                      @update:model-value="set_batch_qty(item, $event)"
                      class="mt-2"
                    >
                      <template v-slot:item="{ props, item }">
                        <v-list-item v-bind="props">
                          <v-list-item-title>{{ item.raw.batch_no }}</v-list-item-title>
                          <v-list-item-subtitle>
                            {{ __("Qty") }}: {{ item.raw.batch_qty }} - {{ __("Expiry") }}: {{ item.raw.expiry_date }}
                          </v-list-item-subtitle>
                        </v-list-item>
                      </template>
                    </v-autocomplete>
                  </v-col>
                  
                  <v-col cols="12" v-if="pos_profile.posa_allow_sales_order && invoiceType == 'Order'">
                    <v-menu v-model="item.item_delivery_date" :close-on-content-click="false">
                      <template v-slot:activator="{ props }">
                        <v-text-field
                          v-model="item.posa_delivery_date"
                          :label="__('Delivery Date')"
                          variant="outlined"
                          density="compact"
                          readonly
                          hide-details
                          v-bind="props"
                          class="mt-2"
                        />
                      </template>
                      <v-date-picker
                        v-model="item.posa_delivery_date"
                        color="primary"
                        :min="frappe.datetime.now_date()"
                        @update:model-value="[item.item_delivery_date = false, validate_due_date(item)]"
                      />
                    </v-menu>
                  </v-col>
                </v-row>

                <div class="item-stats">
                  <div class="stat-item">
                    <span class="stat-label">{{ __("Stock") }}</span>
                    <span class="stat-value">{{ formatFloat(item.actual_qty || 0) }}</span>
                  </div>
                  <div class="stat-item" v-if="item.discount_percentage > 0">
                    <span class="stat-label">{{ __("Discount") }}</span>
                    <span class="stat-value">{{ formatFloat(item.discount_percentage) }}%</span>
                  </div>
                </div>
              </div>
            </v-collapse-transition>
          </div>
        </div>
      </div>

      <div class="footer-section">
        <div class="totals-container">
          <div class="total-item">
            <span class="total-label">{{ __("Items") }}</span>
            <span class="total-value">{{ formatFloat(total_qty) }}</span>
          </div>
          <div class="total-item">
            <span class="total-label">{{ __("Discount") }}</span>
            <span class="total-value">{{ formatCurrency(discount_amount) }}</span>
          </div>
          <div class="total-item total-final">
            <span class="total-label">{{ __("Total") }}</span>
            <span class="total-value">{{ formatCurrency(subtotal) }}</span>
          </div>
        </div>

        <div class="action-buttons">
          <div class="secondary-actions">
            <v-btn
              color="primary"
              variant="outlined"
              size="small"
              @click="save_invoice"
              class="action-btn"
            >
              <v-icon size="small" start>mdi-content-save</v-icon>
              <span class="btn-text">
                {{ __("Save") }}
                <span class="shortcut-key">F10</span>
              </span>
            </v-btn>
            
            <v-btn
              color="orange"
              variant="outlined"
              size="small"
              @click="get_draft_invoices"
              class="action-btn"
            >
              <v-icon size="small" start>mdi-file-document</v-icon>
              <span class="btn-text">
                {{ __("Drafts") }}
                <span class="shortcut-key">F8</span>
              </span>
            </v-btn>
            
            <v-btn
              v-if="pos_profile.custom_allow_select_sales_order === 1"
              color="info"
              variant="outlined"
              size="small"
              @click="get_draft_orders"
              class="action-btn"
            >
              <v-icon size="small" start>mdi-cart</v-icon>
              <span class="btn-text">
                {{ __("Orders") }}
                <span class="shortcut-key">F7</span>
              </span>
            </v-btn>
            
            <v-btn
              color="error"
              variant="outlined"
              size="small"
              @click="cancel_dialog = true"
              class="action-btn"
            >
              <v-icon size="small" start>mdi-close</v-icon>
              <span class="btn-text">
                {{ __("Cancel") }}
                <span class="shortcut-key">F9</span>
              </span>
            </v-btn>
            
            <v-btn
              v-if="pos_profile.posa_allow_return == 1"
              color="secondary"
              variant="outlined"
              size="small"
              @click="open_returns"
              class="action-btn"
            >
              <v-icon size="small" start>mdi-keyboard-return</v-icon>
              <span class="btn-text">
                {{ __("Return") }}
                <span class="shortcut-key">F6</span>
              </span>
            </v-btn>
            
            <v-btn
              v-if="pos_profile.posa_allow_print_draft_invoices"
              color="purple"
              variant="outlined"
              size="small"
              @click="print_draft_invoice"
              class="action-btn"
            >
              <v-icon size="small" start>mdi-printer</v-icon>
              <span class="btn-text">
                {{ __("Print") }}
                <span class="shortcut-key">F5</span>
              </span>
            </v-btn>
          </div>
          
          <v-btn
            color="success"
            variant="flat"
            size="large"
            @click="show_payment"
            class="pay-btn"
          >
            <v-icon size="large" start>mdi-cash</v-icon>
            <span class="btn-text">
              {{ __("PAY") }}
              <span class="shortcut-key pay-shortcut">F12</span>
            </span>
          </v-btn>
        </div>
      </div>
    </v-card>
  </div>
</template>

<script>
import format from "../../format";
import Customer from "./Customer.vue";

export default {
  mixins: [format],
  components: { Customer },
  
  data() {
    return {
      pos_profile: "",
      pos_opening_shift: "",
      stock_settings: "",
      invoice_doc: "",
      return_doc: "",
      customer: "",
      customer_info: "",
      discount_amount: 0,
      additional_discount_percentage: 0,
      total_tax: 0,
      items: [],
      posOffers: [],
      posa_offers: [],
      posa_coupons: [],
      allItems: [],
      discount_percentage_offer_name: null,
      invoiceTypes: ["Invoice", "Order"],
      invoiceType: "Invoice",
      cancel_dialog: false,
      float_precision: 2,
      currency_precision: 2,
      new_line: false,
      delivery_charges: [],
      delivery_charges_rate: 0,
      selected_delivery_charge: "",
      invoice_posting_date: false,
      posting_date: frappe.datetime.nowdate(),
      readonly: false,
      isRTL: false,
      expandedItems: new Set(),
      tempPriceValues: {},
    };
  },

  computed: {
    total_qty() {
      this.close_payments();
      let qty = 0;
      this.items.forEach((item) => {
        qty += parseFloat(item.qty) || 0;
      });
      return this.flt(qty, this.float_precision);
    },

    Total() {
      let sum = 0;
      this.items.forEach((item) => {
        sum += parseFloat(item.qty) * parseFloat(item.rate);
      });
      return this.flt(sum, this.currency_precision);
    },

    subtotal() {
      this.close_payments();
      let sum = 0;
      this.items.forEach((item) => {
        sum += parseFloat(item.qty) * parseFloat(item.rate);
      });
      sum -= this.flt(this.discount_amount);
      sum += this.flt(this.delivery_charges_rate);
      return this.flt(sum, this.currency_precision);
    },

    total_items_discount_amount() {
      let sum = 0;
      this.items.forEach((item) => {
        sum += parseFloat(item.qty) * parseFloat(item.discount_amount);
      });
      return this.flt(sum, this.float_precision);
    },
  },

  methods: {
    formatCurrency(value) {
      const num = parseFloat(value) || 0;
      return num.toFixed(2);
    },

    formatPrice(value) {
      const num = parseFloat(value) || 0;
      return num.toFixed(2);
    },

    handlePriceInputTemp(item, event) {
      let value = event.target ? event.target.value : event;
      value = value.toString().replace(',', '.');
      value = value.replace(/[^0-9.]/g, '');
      
      const parts = value.split('.');
      if (parts.length > 2) {
        value = parts[0] + '.' + parts.slice(1).join('');
      }
      
      this.tempPriceValues[item.posa_row_id] = value;
    },

    handlePriceCommit(item) {
      const tempValue = this.tempPriceValues[item.posa_row_id];
      if (tempValue !== undefined) {
        if (tempValue === '' || tempValue === '.') {
          item.rate = 0;
        } else {
          const numValue = parseFloat(tempValue);
          if (!isNaN(numValue)) {
            item.rate = numValue;
          }
        }
        delete this.tempPriceValues[item.posa_row_id];
        this.handleRateChange(item);
      }
    },

    handlePriceInput(item, event) {
      let value = event.target ? event.target.value : event;
      value = value.toString().replace(',', '.');
      value = value.replace(/[^0-9.]/g, '');
      
      const parts = value.split('.');
      if (parts.length > 2) {
        value = parts[0] + '.' + parts.slice(1).join('');
      }
      
      if (value === '' || value === '.') {
        item.rate = 0;
      } else {
        const numValue = parseFloat(value);
        if (!isNaN(numValue)) {
          item.rate = numValue;
        }
      }
      
      this.handleRateChange(item);
    },

    formatPriceOnBlur(item) {
      item.rate = parseFloat(item.rate) || 0;
      this.$forceUpdate();
    },

    detectRTL() {
      const lang = frappe.boot.lang || 'en';
      this.isRTL = ['ar', 'he', 'fa', 'ur'].includes(lang);
    },

    toggleItemDetails(item) {
      if (this.expandedItems.has(item.posa_row_id)) {
        this.expandedItems.delete(item.posa_row_id);
      } else {
        this.expandedItems.add(item.posa_row_id);
      }
      this.$forceUpdate();
    },

    isItemExpanded(item) {
      return this.expandedItems.has(item.posa_row_id);
    },

    handleKeyboardShortcuts(event) {
      if (event.key === 'F10') {
        event.preventDefault();
        this.save_invoice();
      }
      else if (event.key === 'F9') {
        event.preventDefault();
        if (this.cancel_dialog) {
          this.cancel_dialog = false;
        } else {
          this.cancel_dialog = true;
        }
      }
      else if (event.key === 'F8') {
        event.preventDefault();
        this.get_draft_invoices();
      }
      else if (event.key === 'F7' && this.pos_profile.custom_allow_select_sales_order === 1) {
        event.preventDefault();
        this.get_draft_orders();
      }
      else if (event.key === 'F6' && this.pos_profile.posa_allow_return == 1) {
        event.preventDefault();
        this.open_returns();
      }
      else if (event.key === 'F5' && this.pos_profile.posa_allow_print_draft_invoices) {
        event.preventDefault();
        this.print_draft_invoice();
      }
      else if (event.key === 'F12') {
        event.preventDefault();
        this.show_payment();
      }
    },

    handleQtyChange(item) {
      const newQty = parseFloat(item.qty) || 0;
      
      if (newQty <= 0) {
        this.remove_item(item);
        return;
      }
      
      this.calc_stock_qty(item, newQty);
      this.update_item_detail(item);
      
      if (item.has_serial_no && item.serial_no_selected) {
        if (item.serial_no_selected.length !== newQty) {
          this.eventBus.emit("show_message", {
            text: __("Serial numbers count ({0}) does not match quantity ({1})", [item.serial_no_selected.length, newQty]),
            color: "warning",
          });
          item.qty = item.serial_no_selected.length;
        }
      }
      
      if (item.has_batch_no && item.actual_batch_qty && newQty > item.actual_batch_qty) {
        this.eventBus.emit("show_message", {
          text: __("Requested quantity ({0}) exceeds available batch quantity ({1})", [newQty, item.actual_batch_qty]),
          color: "warning",
        });
        item.qty = item.actual_batch_qty;
      }
      
      this.$forceUpdate();
    },

    handleRateChange(item) {
      let value = item.rate;
      
      if (typeof value === 'string') {
        value = value.replace(',', '.');
      }
      
      const newRate = parseFloat(value);
      
      if (!isNaN(newRate) && newRate >= 0) {
        item.rate = newRate;
        
        if (!item.price_list_rate || item.price_list_rate === 0) {
          item.price_list_rate = newRate;
        }
        
        this.recalculateDiscounts(item);
        this.$forceUpdate();
      }
    },

    recalculateDiscounts(item) {
      const rate = parseFloat(item.rate) || 0;
      const priceListRate = parseFloat(item.price_list_rate) || 0;
      
      if (priceListRate > 0 && rate < priceListRate) {
        const discountAmount = priceListRate - rate;
        const discountPercentage = (discountAmount / priceListRate) * 100;
        
        this.$set(item, 'discount_amount', parseFloat(discountAmount.toFixed(this.currency_precision)));
        this.$set(item, 'discount_percentage', parseFloat(discountPercentage.toFixed(2)));
      } else {
        this.$set(item, 'discount_amount', 0);
        this.$set(item, 'discount_percentage', 0);
        
        if (rate > priceListRate) {
          this.$set(item, 'price_list_rate', rate);
        }
      }
    },

    remove_item(item) {
      const index = this.items.findIndex((el) => el.posa_row_id == item.posa_row_id);
      if (index >= 0) {
        this.items.splice(index, 1);
      }
      
      this.expandedItems.delete(item.posa_row_id);
    },

    add_one(item) {
      item.qty++;
      if (item.qty == 0) {
        this.remove_item(item);
      }
      this.calc_stock_qty(item, item.qty);
      this.update_item_detail(item);
      this.$forceUpdate();
    },

    subtract_one(item) {
      item.qty--;
      if (item.qty == 0) {
        this.remove_item(item);
      }
      this.calc_stock_qty(item, item.qty);
      this.update_item_detail(item);
      this.$forceUpdate();
    },

    add_item(item) {
      if (!item.uom) {
        item.uom = item.stock_uom;
      }
      let index = -1;
      if (!this.new_line) {
        index = this.items.findIndex(
          (el) =>
            el.item_code === item.item_code &&
            el.uom === item.uom &&
            !el.posa_is_offer &&
            !el.posa_is_replace &&
            el.batch_no === item.batch_no
        );
      }
      if (index === -1 || this.new_line) {
        const new_item = this.get_new_item(item);
        this.items.unshift(new_item);
        this.update_item_detail(new_item);
      } else {
        const cur_item = this.items[index];
        cur_item.qty += item.qty || 1;
        this.calc_stock_qty(cur_item, cur_item.qty);
      }
      this.$forceUpdate();
    },

    get_new_item(item) {
      const new_item = { ...item };
      if (!item.qty) {
        item.qty = 1;
      }
      new_item.stock_qty = item.qty;
      new_item.discount_amount = 0;
      new_item.discount_percentage = 0;
      new_item.price_list_rate = item.rate || 0;
      new_item.qty = item.qty;
      new_item.uom = item.uom ? item.uom : item.stock_uom;
      new_item.conversion_factor = 1;
      new_item.posa_is_offer = item.posa_is_offer || 0;
      new_item.posa_is_replace = item.posa_is_replace || null;
      new_item.is_free_item = item.is_free_item || 0;
      new_item.posa_notes = "";
      new_item.posa_delivery_date = "";
      new_item.posa_row_id = this.makeid(20);
      
      if (!new_item.item_uoms || !Array.isArray(new_item.item_uoms)) {
        new_item.item_uoms = [{
          uom: new_item.stock_uom || new_item.uom || 'Nos',
          conversion_factor: 1
        }];
      }
      
      if (new_item.is_free_item || new_item.posa_is_offer) {
        new_item.rate = 0;
        new_item.price_list_rate = 0;
      }
      
      return new_item;
    },

    clear_invoice() {
      this.items = [];
      this.posa_offers = [];
      this.posa_coupons = [];
      this.expandedItems.clear();
      this.tempPriceValues = {};
      this.customer = this.pos_profile.customer;
      this.invoice_doc = "";
      this.return_doc = "";
      this.discount_amount = 0;
      this.additional_discount_percentage = 0;
      this.delivery_charges_rate = 0;
      this.selected_delivery_charge = "";
      this.eventBus.emit("set_customer_readonly", false);
      this.invoiceType = this.pos_profile.posa_default_sales_order ? "Order" : "Invoice";
      this.invoiceTypes = ["Invoice", "Order"];
    },

    async cancel_invoice() {
      const doc = this.get_invoice_doc();
      this.invoiceType = this.pos_profile.posa_default_sales_order ? "Order" : "Invoice";
      this.invoiceTypes = ["Invoice", "Order"];
      this.posting_date = frappe.datetime.nowdate();
      
      if (doc.name && this.pos_profile.posa_allow_delete) {
        await frappe.call({
          method: "posawesome.posawesome.api.posapp.delete_invoice",
          args: { invoice: doc.name },
          async: true,
          callback: (r) => {
            if (r.message) {
              this.eventBus.emit("show_message", {
                text: r.message,
                color: "warning",
              });
            }
          },
        });
      }
      this.clear_invoice();
      this.cancel_dialog = false;
    },

    save_invoice() {
      const doc = this.get_invoice_doc();
      let saved_invoice;
      
      if (doc.name) {
        saved_invoice = this.update_invoice(doc);
      } else {
        if (doc.items.length) {
          saved_invoice = this.update_invoice(doc);
        } else {
          this.eventBus.emit("show_message", {
            title: __("Nothing to save"),
            color: "error",
          });
          return;
        }
      }
      
      if (!saved_invoice) {
        this.eventBus.emit("show_message", {
          title: __("Error saving the invoice"),
          color: "error",
        });
      } else {
        this.eventBus.emit("show_message", {
          title: __("Invoice saved successfully"),
          color: "success",
        });
        this.clear_invoice();
        return saved_invoice;
      }
    },

    save_and_clear_invoice() {
      const saved_invoice = this.save_invoice();
      if (saved_invoice) {
        this.clear_invoice();
      }
      return saved_invoice;
    },

    async load_invoice(data = {}) {
      this.clear_invoice();
      if (data.is_return) {
        this.eventBus.emit("set_customer_readonly", true);
        this.invoiceType = "Return";
        this.invoiceTypes = ["Return"];
      }
      this.invoice_doc = data;
      this.items = data.items;
      this.posa_offers = data.posa_offers || [];
      this.items.forEach((item) => {
        if (!item.posa_row_id) {
          item.posa_row_id = this.makeid(20);
        }
      });
      this.customer = data.customer;
      this.posting_date = data.posting_date || frappe.datetime.nowdate();
      this.discount_amount = data.discount_amount;
      this.additional_discount_percentage = data.additional_discount_percentage;
    },

    async load_return_invoice(data = {}) {
      this.clear_invoice();
      if (data.items && data.items.length > 0) {
        this.eventBus.emit("set_customer_readonly", true);
        this.invoiceType = "Return";
        this.invoiceTypes = ["Return"];
        this.invoice_doc = {
          is_return: 1,
          return_against: data.name,
          customer: data.customer,
          posting_date: frappe.datetime.nowdate()
        };
        this.return_doc = data;
        
        const return_items = [];
        data.items.forEach((item) => {
          const return_item = {
            ...item,
            qty: -Math.abs(item.qty),
            stock_qty: -Math.abs(item.stock_qty),
            posa_row_id: this.makeid(20)
          };
          return_items.push(return_item);
        });
        
        this.items = return_items;
        this.customer = data.customer;
        this.posting_date = frappe.datetime.nowdate();
        this.discount_amount = data.discount_amount || 0;
        this.additional_discount_percentage = data.additional_discount_percentage || 0;
      }
    },

    get_invoice_doc() {
      let doc = {};
      if (this.invoice_doc.name) {
        doc = { ...this.invoice_doc };
      }
      doc.doctype = "Sales Invoice";
      doc.is_pos = 1;
      doc.ignore_pricing_rule = 1;
      doc.company = doc.company || this.pos_profile.company;
      doc.pos_profile = doc.pos_profile || this.pos_profile.name;
      doc.campaign = doc.campaign || this.pos_profile.campaign;
      doc.currency = doc.currency || this.pos_profile.currency;
      doc.naming_series = doc.naming_series || this.pos_profile.naming_series;
      doc.customer = this.customer;
      doc.items = this.get_invoice_items();
      doc.total = this.subtotal;
      doc.discount_amount = parseFloat(this.discount_amount) || 0;
      doc.additional_discount_percentage = parseFloat(this.additional_discount_percentage) || 0;
      doc.posa_pos_opening_shift = this.pos_opening_shift.name;
      doc.payments = this.get_payments();
      doc.taxes = [];
      doc.is_return = this.invoice_doc.is_return || 0;
      doc.return_against = this.invoice_doc.return_against || "";
      doc.posa_offers = this.posa_offers;
      doc.posa_coupons = this.posa_coupons;
      doc.posa_delivery_charges = this.selected_delivery_charge.name;
      doc.posa_delivery_charges_rate = this.delivery_charges_rate || 0;
      doc.posting_date = this.posting_date;
      return doc;
    },

    get_invoice_items() {
      const items_list = [];
      this.items.forEach((item) => {
        const new_item = {
          item_code: item.item_code,
          posa_row_id: item.posa_row_id,
          posa_offers: item.posa_offers,
          posa_offer_applied: item.posa_offer_applied,
          posa_is_offer: item.posa_is_offer,
          posa_is_replace: item.posa_is_replace,
          is_free_item: item.is_free_item,
          qty: parseFloat(item.qty) || 0,
          rate: parseFloat(item.rate) || 0,
          uom: item.uom,
          amount: (parseFloat(item.qty) || 0) * (parseFloat(item.rate) || 0),
          conversion_factor: item.conversion_factor,
          serial_no: item.serial_no,
          discount_percentage: parseFloat(item.discount_percentage) || 0,
          discount_amount: parseFloat(item.discount_amount) || 0,
          batch_no: item.batch_no,
          posa_notes: item.posa_notes,
          posa_delivery_date: item.posa_delivery_date,
          price_list_rate: item.price_list_rate,
        };
        items_list.push(new_item);
      });
      return items_list;
    },

    get_payments() {
      const payments = [];
      this.pos_profile.payments.forEach((payment) => {
        payments.push({
          amount: 0,
          mode_of_payment: payment.mode_of_payment,
          default: payment.default,
          account: "",
        });
      });
      return payments;
    },

    update_invoice(doc) {
      var vm = this;
      frappe.call({
        method: "posawesome.posawesome.api.posapp.update_invoice",
        args: {
          data: doc,
        },
        async: false,
        callback: function (r) {
          if (r.message) {
            vm.invoice_doc = r.message;
          }
        },
      });
      return this.invoice_doc;
    },

    process_invoice() {
      const doc = this.get_invoice_doc();
      if (doc.name) {
        return this.update_invoice(doc);
      } else {
        return this.update_invoice(doc);
      }
    },

    async show_payment() {
      if (!this.customer) {
        this.eventBus.emit("show_message", {
          title: __("Select a customer"),
          color: "error",
        });
        return;
      }
      if (!this.items.length) {
        this.eventBus.emit("show_message", {
          title: __("Select items to sell"),
          color: "error",
        });
        return;
      }
      if (!this.validate()) {
        return;
      }

      this.eventBus.emit("show_payment", "true");
      const invoice_doc = this.process_invoice();
      this.eventBus.emit("send_invoice_doc_payment", invoice_doc);
    },

    validate() {
      let value = true;
      var vm = this;
      this.items.forEach((item) => {
        if (item.qty == 0) {
          vm.eventBus.emit("show_message", {
            title: __("Quantity for item '{0}' cannot be Zero (0)", [
              item.item_name,
            ]),
            color: "error",
          });
          value = false;
        }
        if (parseFloat(item.rate) <= 0 && !item.is_free_item && !item.posa_is_offer) {
          vm.eventBus.emit("show_message", {
            title: __("Rate cannot be zero for item '{0}'", [
              item.item_name,
            ]),
            color: "error",
          });
          value = false;
        }
      });
      return value;
    },

    get_draft_invoices() {
      var vm = this;
      frappe.call({
        method: "posawesome.posawesome.api.posapp.get_draft_invoices",
        args: {
          pos_opening_shift: this.pos_opening_shift.name,
        },
        async: false,
        callback: function (r) {
          if (r.message && Array.isArray(r.message)) {
            vm.eventBus.emit("open_drafts", r.message);
          } else {
            vm.eventBus.emit("show_message", {
              title: __("No draft invoices found"),
              color: "info",
            });
          }
        },
        error: function(r) {
          vm.eventBus.emit("show_message", {
            title: __("Error loading draft invoices"),
            color: "error",
          });
        }
      });
    },

    get_draft_orders() {
      var vm = this;
      frappe.call({
        method: "posawesome.posawesome.api.posapp.search_orders",
        args: {
          company: this.pos_profile.company,
          currency: this.pos_profile.currency,
        },
        async: false,
        callback: function (r) {
          if (r.message) {
            vm.eventBus.emit("open_orders", r.message);
          }
        },
      });
    },

    open_returns() {
      this.eventBus.emit("open_returns", this.pos_profile.company);
    },

    close_payments() {
      this.eventBus.emit("show_payment", "false");
    },

    update_item_detail(item) {
      if (!item.item_code || this.invoice_doc.is_return) {
        return;
      }
      var vm = this;
      frappe.call({
        method: "posawesome.posawesome.api.posapp.get_item_detail",
        args: {
          warehouse: this.pos_profile.warehouse,
          price_list: this.pos_profile.price_list,
          item: {
            item_code: item.item_code,
            customer: this.customer,
            doctype: "Sales Invoice",
            name: "New Sales Invoice 1",
            company: this.pos_profile.company,
            conversion_rate: 1,
            qty: item.qty,
            price_list_rate: item.price_list_rate,
            currency: this.pos_profile.currency,
            pos_profile: this.pos_profile.name,
            uom: item.uom,
            update_stock: this.pos_profile.update_stock,
            price_list: this.get_price_list(),
            has_batch_no: item.has_batch_no,
            serial_no: item.serial_no,
            batch_no: item.batch_no,
            is_stock_item: item.is_stock_item,
          },
        },
        callback: function (r) {
          if (r.message) {
            const data = r.message;
            if (!item.is_free_item && !item.posa_is_offer && !item.posa_is_replace) {
              item.price_list_rate = data.price_list_rate;
            }
            item.last_purchase_rate = data.last_purchase_rate;
            item.projected_qty = data.projected_qty;
            item.reserved_qty = data.reserved_qty;
            item.conversion_factor = data.conversion_factor;
            item.stock_qty = data.stock_qty;
            item.actual_qty = data.actual_qty;
            item.stock_uom = data.stock_uom;
            item.has_serial_no = data.has_serial_no;
            item.has_batch_no = data.has_batch_no;
            
            if (data.item_uoms && Array.isArray(data.item_uoms)) {
              item.item_uoms = data.item_uoms;
            } else {
              item.item_uoms = [{
                uom: item.stock_uom || item.uom,
                conversion_factor: 1
              }];
            }
            
            vm.calc_item_price(item);
          }
        },
        error: function(r) {
          console.error('Error updating item detail:', r);
        }
      });
    },

    fetch_customer_details() {
      var vm = this;
      if (this.customer) {
        frappe.call({
          method: "posawesome.posawesome.api.posapp.get_customer_info",
          args: {
            customer: vm.customer,
          },
          async: false,
          callback: (r) => {
            const message = r.message;
            if (!r.exc) {
              vm.customer_info = {
                ...message,
              };
            }
            vm.update_price_list();
          },
        });
      }
    },

    get_price_list() {
      let price_list = this.pos_profile.selling_price_list;
      if (this.customer_info && this.pos_profile) {
        const { customer_price_list, customer_group_price_list } = this.customer_info;
        const pos_price_list = this.pos_profile.selling_price_list;
        if (customer_price_list && customer_price_list != pos_price_list) {
          price_list = customer_price_list;
        } else if (customer_group_price_list && customer_group_price_list != pos_price_list) {
          price_list = customer_group_price_list;
        }
      }
      return price_list;
    },

    update_price_list() {
      let price_list = this.get_price_list();
      if (price_list == this.pos_profile.selling_price_list) {
        price_list = null;
      }
      this.eventBus.emit("update_customer_price_list", price_list);
    },

    update_discount_umount() {
      const value = parseFloat(this.additional_discount_percentage) || 0;
      if (value >= -100 && value <= 100) {
        this.discount_amount = (this.Total * value) / 100;
      } else {
        this.additional_discount_percentage = 0;
        this.discount_amount = 0;
      }
    },

    calc_item_price(item) {
      if (!item.posa_offer_applied) {
        if (item.price_list_rate) {
          item.rate = item.price_list_rate;
        }
      }
      if (item.discount_percentage) {
        item.rate =
          parseFloat(item.price_list_rate) -
          (parseFloat(item.price_list_rate) * parseFloat(item.discount_percentage)) / 100;
        item.discount_amount = this.flt(
          parseFloat(item.price_list_rate) - parseFloat(item.rate),
          this.currency_precision
        );
      } else if (item.discount_amount) {
        item.rate = this.flt(
          parseFloat(item.price_list_rate) - parseFloat(item.discount_amount),
          this.currency_precision
        );
      }
    },

    calc_uom(item, value) {
      if (!item.item_uoms || item.item_uoms.length === 0) {
        console.warn('No UOMs available for item:', item.item_code);
        return;
      }
      
      const new_uom = item.item_uoms.find((element) => element.uom == value);
      if (!new_uom) {
        console.warn('UOM not found:', value);
        return;
      }
      
      item.conversion_factor = new_uom.conversion_factor;
      if (!item.posa_offer_applied) {
        item.discount_amount = 0;
        item.discount_percentage = 0;
      }
      this.update_item_detail(item);
    },

    calc_stock_qty(item, value) {
      item.stock_qty = item.conversion_factor * value;
    },

    set_serial_no(item) {
      if (!item.has_serial_no) return;
      item.serial_no = "";
      item.serial_no_selected.forEach((element) => {
        item.serial_no += element + "\n";
      });
      item.serial_no_selected_count = item.serial_no_selected.length;
      if (item.serial_no_selected_count != item.stock_qty) {
        item.qty = item.serial_no_selected_count;
        this.calc_stock_qty(item, item.qty);
        this.$forceUpdate();
      }
    },

    set_batch_qty(item, value, update = true) {
      const existing_items = this.items.filter(
        (element) =>
          element.item_code == item.item_code &&
          element.posa_row_id != item.posa_row_id
      );
      const used_batches = {};
      item.batch_no_data.forEach((batch) => {
        used_batches[batch.batch_no] = {
          ...batch,
          used_qty: 0,
          remaining_qty: batch.batch_qty,
        };
        existing_items.forEach((element) => {
          if (element.batch_no && element.batch_no == batch.batch_no) {
            used_batches[batch.batch_no].used_qty += element.qty;
            used_batches[batch.batch_no].remaining_qty -= element.qty;
            used_batches[batch.batch_no].batch_qty -= element.qty;
          }
        });
      });

      const batch_no_data = Object.values(used_batches)
        .filter((batch) => batch.remaining_qty > 0)
        .sort((a, b) => {
          if (a.expiry_date && b.expiry_date) {
            return a.expiry_date - b.expiry_date;
          } else if (a.expiry_date) {
            return -1;
          } else if (b.expiry_date) {
            return 1;
          } else {
            return b.remaining_qty - a.remaining_qty;
          }
        });

      if (batch_no_data.length > 0) {
        let batch_to_use = null;
        if (value) {
          batch_to_use = batch_no_data.find((batch) => batch.batch_no == value);
        }
        if (!batch_to_use) {
          batch_to_use = batch_no_data[0];
        }
        item.batch_no = batch_to_use.batch_no;
        item.actual_batch_qty = batch_to_use.batch_qty;
        item.batch_no_expiry_date = batch_to_use.expiry_date;
        if (update) {
          this.update_item_detail(item);
        }
      } else {
        item.batch_no = null;
        item.actual_batch_qty = null;
        item.batch_no_expiry_date = null;
      }
      item.batch_no_data = batch_no_data;
    },

    validate_due_date(item) {
      const today = frappe.datetime.now_date();
      const parse_today = Date.parse(today);
      const new_date = Date.parse(item.posa_delivery_date);
      if (new_date < parse_today) {
        setTimeout(() => {
          item.posa_delivery_date = today;
        }, 0);
      }
    },

    update_delivery_charges() {
      if (this.selected_delivery_charge) {
        this.delivery_charges_rate = this.selected_delivery_charge.rate;
      } else {
        this.delivery_charges_rate = 0;
      }
    },

    set_delivery_charges() {
      var vm = this;
      if (!this.pos_profile || !this.customer || !this.pos_profile.posa_use_delivery_charges) {
        this.delivery_charges = [];
        this.delivery_charges_rate = 0;
        this.selected_delivery_charge = "";
        return;
      }
      this.delivery_charges_rate = 0;
      this.selected_delivery_charge = "";
      frappe.call({
        method: "posawesome.posawesome.api.posapp.get_applicable_delivery_charges",
        args: {
          company: this.pos_profile.company,
          pos_profile: this.pos_profile.name,
          customer: this.customer,
        },
        async: false,
        callback: function (r) {
          if (r.message) {
            if (r.message?.length) {
              vm.delivery_charges = r.message;
            }
          }
        },
      });
    },

    print_draft_invoice() {
      if (!this.pos_profile.posa_allow_print_draft_invoices) {
        this.eventBus.emit("show_message", {
          title: __("You are not allowed to print draft invoices"),
          color: "error",
        });
        return;
      }
      let invoice_name = this.invoice_doc.name;
      frappe.run_serially([
        () => {
          const invoice_doc = this.save_and_clear_invoice();
          invoice_name = invoice_doc.name ? invoice_doc.name : invoice_name;
        },
        () => {
          this.load_print_page(invoice_name);
        },
      ]);
    },

    load_print_page(invoice_name) {
      const print_format = this.pos_profile.print_format_for_online || this.pos_profile.print_format;
      const letter_head = this.pos_profile.letter_head || 0;
      const url = frappe.urllib.get_base_url() + 
        "/printview?doctype=Sales%20Invoice&name=" + invoice_name + 
        "&trigger_print=1&format=" + print_format + 
        "&no_letterhead=" + letter_head;
      const printWindow = window.open(url, "Print");
      printWindow.addEventListener("load", function () {
        printWindow.print();
      }, true);
    },

    makeid(length) {
      let result = "";
      const characters = "abcdefghijklmnopqrstuvwxyz0123456789";
      const charactersLength = characters.length;
      for (var i = 0; i < length; i++) {
        result += characters.charAt(Math.floor(Math.random() * charactersLength));
      }
      return result;
    },

    isNumber(value) {
      return !isNaN(parseFloat(value)) && isFinite(value);
    },

    setFormatedFloat(target, field, precision, allowNegative, value) {
      if (typeof target === 'object') {
        target[field] = parseFloat(value).toFixed(precision || this.float_precision);
      } else {
        this[target] = parseFloat(value).toFixed(precision || this.currency_precision);
      }
    },

    setFormatedCurrency(target, field, precision, allowNegative, value) {
      if (typeof target === 'object') {
        target[field] = parseFloat(value).toFixed(precision || this.currency_precision);
      } else {
        this[target] = parseFloat(value).toFixed(precision || this.currency_precision);
      }
    },
  },

  mounted() {
    this.detectRTL();
    
    window.addEventListener('keydown', this.handleKeyboardShortcuts);
    
    this.eventBus.on("register_pos_profile", (data) => {
      this.pos_profile = data.pos_profile;
      this.customer = data.pos_profile.customer;
      this.pos_opening_shift = data.pos_opening_shift;
      this.stock_settings = data.stock_settings;
      this.float_precision = frappe.defaults.get_default("float_precision") || 2;
      this.currency_precision = frappe.defaults.get_default("currency_precision") || 2;
      this.invoiceType = this.pos_profile.posa_default_sales_order ? "Order" : "Invoice";
    });

    this.eventBus.on("add_item", (item) => {
      this.add_item(item);
    });

    this.eventBus.on("update_customer", (customer) => {
      this.customer = customer;
    });

    this.eventBus.on("fetch_customer_details", () => {
      this.fetch_customer_details();
    });

    this.eventBus.on("clear_invoice", () => {
      this.clear_invoice();
    });

    this.eventBus.on("load_invoice", (data) => {
      this.load_invoice(data);
    });

    this.eventBus.on("load_order", (data) => {
      this.load_invoice(data);
    });

    this.eventBus.on("load_return_invoice", (data) => {
      this.load_return_invoice(data);
    });

    this.eventBus.on("set_new_line", (data) => {
      this.new_line = data;
    });

    this.eventBus.on("set_all_items", (data) => {
      this.allItems = data;
      this.items.forEach((item) => {
        this.update_item_detail(item);
      });
    });
  },

  beforeUnmount() {
    window.removeEventListener('keydown', this.handleKeyboardShortcuts);
    
    this.eventBus.$off("register_pos_profile");
    this.eventBus.$off("add_item");
    this.eventBus.$off("update_customer");
    this.eventBus.$off("fetch_customer_details");
    this.eventBus.$off("clear_invoice");
    this.eventBus.$off("load_invoice");
    this.eventBus.$off("load_order");
    this.eventBus.$off("load_return_invoice");
    this.eventBus.$off("set_new_line");
    this.eventBus.$off("set_all_items");
  },

  watch: {
    customer() {
      this.close_payments();
      this.eventBus.emit("set_customer", this.customer);
      this.fetch_customer_details();
      this.set_delivery_charges();
    },

    customer_info() {
      this.eventBus.emit("set_customer_info_to_edit", this.customer_info);
    },

    items: {
      deep: true,
      handler(items) {
        this.$forceUpdate();
      },
    },

    invoiceType() {
      this.eventBus.emit("update_invoice_type", this.invoiceType);
    },

    discount_amount() {
      if (!this.discount_amount || this.discount_amount == 0) {
        this.additional_discount_percentage = 0;
      } else if (this.pos_profile.posa_use_percentage_discount) {
        this.additional_discount_percentage = (this.discount_amount / this.Total) * 100;
      } else {
        this.additional_discount_percentage = 0;
      }
    },
  },
};
</script>

<style scoped>
.invoice-container {
  height: 100%;
  width: 100%;
  background: #f4f5f7;
}

.invoice-card {
  height: 100%;
  display: flex;
  flex-direction: column;
  border-radius: 12px !important;
  overflow: hidden;
}

.header-section {
  background: white;
  border-bottom: 2px solid #e9ecef;
  flex-shrink: 0;
  padding: 8px 0;
}

.items-section {
  flex: 1;
  overflow: hidden;
  background: #f8f9fa;
  padding: 4px;
  position: relative;
}

.empty-cart {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  background: #f0f4f8;
  padding: 20px;
  overflow: hidden;
}

.empty-cart::before {
  content: '';
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  width: 500px;
  height: 500px;
  background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 24 24' fill='none'%3E%3Cpath d='M1 1H5L7.68 14.39C7.77144 14.8504 8.02191 15.264 8.38755 15.5583C8.75318 15.8526 9.2107 16.009 9.68 16H19.4C19.8693 16.009 20.3268 15.8526 20.6925 15.5583C21.0581 15.264 21.3086 14.8504 21.4 14.39L23 6H6M9 21C9 21.5523 8.55228 22 8 22C7.44772 22 7 21.5523 7 21C7 20.4477 7.44772 20 8 20C8.55228 20 9 20.4477 9 21ZM20 21C20 21.5523 19.5523 22 19 22C18.4477 22 18 21.5523 18 21C18 20.4477 18.4477 20 19 20C19.5523 20 20 20.4477 20 21Z' stroke='%23e2e8f0' stroke-width='2' stroke-linecap='round' stroke-linejoin='round'/%3E%3C/svg%3E");
  background-repeat: no-repeat;
  background-position: center;
  background-size: contain;
  opacity: 0.15;
  animation: cartMove 20s ease-in-out infinite;
  pointer-events: none;
}

@keyframes cartMove {
  0%, 100% {
    transform: translate(-50%, -50%) rotate(0deg) scale(1);
  }
  25% {
    transform: translate(-45%, -55%) rotate(5deg) scale(1.05);
  }
  50% {
    transform: translate(-55%, -50%) rotate(-5deg) scale(0.95);
  }
  75% {
    transform: translate(-50%, -45%) rotate(3deg) scale(1.02);
  }
}

.empty-cart-content {
  text-align: center;
  max-width: 400px;
  width: 100%;
  padding: 40px;
  background: rgba(255, 255, 255, 0.95);
  border-radius: 20px;
  box-shadow: 0 10px 40px rgba(0, 0, 0, 0.1);
  backdrop-filter: blur(10px);
  transform: translateY(-20px);
  animation: fadeInUp 0.6s ease-out;
  position: relative;
  z-index: 1;
}

@keyframes fadeInUp {
  from {
    opacity: 0;
    transform: translateY(0);
  }
  to {
    opacity: 1;
    transform: translateY(-20px);
  }
}

.empty-cart-icon {
  width: 120px;
  height: 120px;
  margin: 0 auto 24px;
  color: #3b82f6;
  animation: float 3s ease-in-out infinite;
  position: relative;
}

@keyframes float {
  0%, 100% {
    transform: translateY(0);
  }
  50% {
    transform: translateY(-10px);
  }
}

.empty-cart-icon svg {
  width: 100%;
  height: 100%;
  filter: drop-shadow(0 4px 6px rgba(59, 130, 246, 0.2));
}

.empty-cart-title {
  font-size: 1.75rem;
  font-weight: 700;
  color: #334155;
  margin-bottom: 12px;
  letter-spacing: -0.025em;
}

.empty-cart-subtitle {
  font-size: 1rem;
  color: #64748b;
  margin: 0;
  line-height: 1.5;
}

[dir="rtl"] .empty-cart-title,
[dir="rtl"] .empty-cart-subtitle {
  font-family: 'Cairo', 'Segoe UI', Tahoma, sans-serif;
}

.items-list {
  height: 100%;
  overflow-y: auto;
  overflow-x: hidden;
  padding: 2px;
}

.item-card {
  background: white;
  border: 1px solid #e4e7eb;
  border-radius: 6px;
  margin-bottom: 4px;
  transition: all 0.2s ease;
  overflow: hidden;
}

.item-card:hover {
  border-color: #cbd5e1;
  box-shadow: 0 1px 3px rgba(0,0,0,0.1);
}

.item-content {
  padding: 8px 10px;
  background: #fffbee;
}

.item-header-line {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 6px;
  cursor: pointer;
}

.item-title {
  display: flex;
  align-items: center;
  gap: 6px;
  flex: 1;
  min-width: 0;
}

[dir="ltr"] .item-title {
  text-align: left;
}

[dir="rtl"] .item-title {
  flex-direction: row-reverse;
  text-align: right;
}

.item-code {
  font-size: 0.75rem;
  color: #94a3b8;
  font-weight: 400;
  flex-shrink: 0;
}

.separator {
  color: #e2e8f0;
  font-size: 0.8rem;
  flex-shrink: 0;
  margin: 0 6px;
}

.item-name {
  font-size: 0.85rem;
  font-weight: 600;
  color: #1e293b;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  flex: 1;
}

.offer-chip {
  height: 18px !important;
  font-size: 0.65rem !important;
  padding: 0 6px !important;
}

.item-data-line {
  display: flex;
  align-items: center;
  gap: 12px;
}

.data-group {
  display: flex;
  align-items: center;
  background: #f8fafc;
  border: 1px solid #e2e8f0;
  border-radius: 4px;
  padding: 4px 8px;
  min-height: 28px;
}

.qty-group {
  padding: 2px;
  gap: 2px;
  background: white;
}

.qty-btn {
  width: 24px !important;
  height: 24px !important;
  min-width: 24px !important;
  background: #f1f5f9 !important;
  border-radius: 3px !important;
}

.qty-btn:hover {
  background: #e2e8f0 !important;
}

.qty-btn :deep(.v-icon) {
  font-size: 0.75rem !important;
  color: #475569;
}

.qty-input {
  width: 45px;
  text-align: center;
  border: none;
  background: transparent;
  font-size: 0.875rem;
  font-weight: 600;
  color: #0ea5e9;
  outline: none;
  padding: 0 4px;
  -moz-appearance: textfield;
}

.qty-input::-webkit-outer-spin-button,
.qty-input::-webkit-inner-spin-button {
  -webkit-appearance: none;
  margin: 0;
}

.qty-input:focus {
  background: #f0f9ff;
  border-radius: 2px;
}

.uom-group {
  width: 10%;
  justify-content: center;
  padding: 0;
}

.uom-select {
  width: 100%;
  font-size: 0.75rem;
}

.uom-select :deep(.v-field__input) {
  padding: 0 !important;
  min-height: 0 !important;
  font-size: 0.75rem;
  color: #64748b;
  font-weight: 500;
  text-align: center;
}

.uom-select :deep(.v-field__append-inner) {
  padding: 0 !important;
}

.uom-select :deep(.v-field) {
  min-height: 0 !important;
}

.uom-select :deep(.v-select__selection) {
  font-size: 0.75rem;
  color: #64748b;
  font-weight: 500;
}

.uom-select :deep(.v-icon) {
  font-size: 0.875rem !important;
  margin-left: 2px;
}

[dir="rtl"] .uom-select :deep(.v-icon) {
  margin-left: 0;
  margin-right: 2px;
}

.data-label {
  font-size: 0.75rem;
  color: #64748b;
  font-weight: 500;
}

.price-group {
  flex: 1;
  padding: 0;
  background: white;
  width: 20%;
}

.price-input {
  width: 100%;
  text-align: center;
  border: none;
  background: transparent;
  padding: 4px 8px;
  font-size: 0.875rem;
  font-weight: 500;
  color: #059669;
  outline: none;
  -moz-appearance: textfield;
  font-feature-settings: "tnum";
  direction: ltr;
  unicode-bidi: embed;
}

.price-input::-webkit-outer-spin-button,
.price-input::-webkit-inner-spin-button {
  -webkit-appearance: none;
  margin: 0;
}

.price-input:focus {
  background: #f0fdf4;
  border-radius: 3px;
}

.price-input:disabled {
  background: #f8fafc;
  color: #94a3b8;
  cursor: not-allowed;
}

.total-group {
  width: 20%;
  justify-content: flex-end;
  background: #eff6ff;
  border-color: #dbeafe;
}

.total-value {
  font-size: 0.9rem;
  font-weight: 700;
  color: #1e40af;
}

.action-group {
  display: flex;
  align-items: center;
  width: 5%;
}

.delete-btn {
  width: 28px !important;
  height: 28px !important;
  min-width: 28px !important;
}

.delete-btn :deep(.v-icon) {
  font-size: 0.85rem !important;
}

[dir="rtl"] .item-title {
  flex-direction: row-reverse;
}

[dir="rtl"] .item-header-line {
  flex-direction: row-reverse;
}

[dir="rtl"] .item-data-line {
  flex-direction: row-reverse;
}

[dir="rtl"] .qty-group {
  flex-direction: row-reverse;
}

[dir="rtl"] .price-input {
  direction: rtl;
  text-align: center;
}

[dir="rtl"] .total-group {
  justify-content: flex-start;
}

[dir="rtl"] .data-group {
  direction: rtl;
}

@media (max-width: 768px) {
  .item-data-line {
    gap: 8px;
  }
  
  .data-group {
    padding: 3px 6px;
    min-height: 26px;
  }
  
  .qty-input {
    width: 40px;
  }
  
  .price-group {
    min-width: 70px;
  }
  
  .total-group {
    min-width: 80px;
  }
  
  .empty-cart-content {
    padding: 30px 20px;
  }
  
  .empty-cart-icon {
    width: 80px;
    height: 80px;
  }
  
  .empty-cart-title {
    font-size: 1.5rem;
  }
  
  .empty-cart-subtitle {
    font-size: 0.875rem;
  }
}

[dir="rtl"] .item-row-content {
  flex-direction: row-reverse;
}

[dir="rtl"] .item-left-section {
  flex-direction: row-reverse;
}

[dir="rtl"] .item-right-section {
  flex-direction: row-reverse;
}

[dir="rtl"] .unit-price-row,
[dir="rtl"] .total-price-row {
  flex-direction: row-reverse;
}

[dir="rtl"] .qty-controls {
  flex-direction: row-reverse;
}

[dir="rtl"] .item-prices {
  align-items: flex-start;
}

[dir="rtl"] .item-info {
  text-align: right;
}

.item-details {
  padding: 8px 12px;
  background: #f0f4f8;
  border-top: 1px solid #e0e0e0;
}

.detail-field {
  margin-bottom: 8px;
}

.detail-field label {
  display: block;
  font-size: 0.8rem;
  color: #6c757d;
  margin-bottom: 4px;
  font-weight: 500;
}

.compact-input {
  width: 100%;
}

.compact-input :deep(.v-field) {
  min-height: 36px !important;
}

.compact-input :deep(.v-field__input) {
  padding: 0 12px !important;
  font-size: 0.9rem;
}

.item-stats {
  display: flex;
  gap: 16px;
  margin-top: 12px;
  padding-top: 12px;
  border-top: 1px solid #e9ecef;
}

.stat-item {
  display: flex;
  flex-direction: column;
}

.stat-label {
  font-size: 0.75rem;
  color: #6c757d;
  margin-bottom: 2px;
}

.stat-value {
  font-weight: 600;
  color: #495057;
}

.footer-section {
  background: white;
  border-top: 2px solid #e9ecef;
  flex-shrink: 0;
}

.totals-container {
  display: flex;
  justify-content: space-between;
  padding: 8px 12px;
  border-bottom: 1px solid #e9ecef;
  background: #f8f9fa;
}

.total-item {
  display: flex;
  align-items: center;
  gap: 4px;
}

.total-item .total-label {
  font-size: 0.75rem;
  color: #6c757d;
}

.total-item .total-value {
  font-weight: 600;
  font-size: 0.85rem;
  color: #495057;
}

.total-final .total-value {
  font-size: 1rem;
  color: #2e7d32;
  font-weight: 700;
}

.action-buttons {
  padding: 8px;
}

.secondary-actions {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(80px, 1fr));
  gap: 6px;
  margin-bottom: 8px;
}

.action-btn {
  font-size: 0.75rem !important;
  text-transform: none !important;
  padding: 0 6px !important;
  height: 32px !important;
  position: relative;
}

.action-btn :deep(.v-icon) {
  font-size: 0.85rem !important;
}

.btn-text {
  display: flex;
  align-items: center;
  gap: 4px;
}

.shortcut-key {
  font-size: 0.65rem;
  background: rgba(0, 0, 0, 0.1);
  padding: 1px 4px;
  border-radius: 3px;
  margin-left: 4px;
  font-weight: 600;
  opacity: 0.8;
}

[dir="rtl"] .shortcut-key {
  margin-left: 0;
  margin-right: 4px;
}

.pay-btn {
  width: 100%;
  height: 44px !important;
  font-size: 1rem !important;
  font-weight: 600 !important;
  text-transform: uppercase !important;
  letter-spacing: 0.5px !important;
  box-shadow: 0 2px 6px rgba(76, 175, 80, 0.3) !important;
}

.pay-btn:hover {
  box-shadow: 0 4px 12px rgba(76, 175, 80, 0.4) !important;
  transform: translateY(-1px);
}

.pay-btn :deep(.v-icon) {
  font-size: 1.2rem !important;
}

.pay-shortcut {
  background: rgba(255, 255, 255, 0.2);
  font-size: 0.75rem;
  margin-left: 8px;
}

[dir="rtl"] .pay-shortcut {
  margin-left: 0;
  margin-right: 8px;
}

[dir="rtl"] .item-header {
  flex-direction: row-reverse;
}

[dir="rtl"] .summary-row {
  flex-direction: row-reverse;
}

[dir="rtl"] .qty-controls {
  flex-direction: row-reverse;
}

[dir="rtl"] .price-info {
  flex-direction: row-reverse;
}

[dir="rtl"] .total-row {
  flex-direction: row-reverse;
}

[dir="rtl"] .totals-container {
  flex-direction: row-reverse;
}

[dir="rtl"] .secondary-actions {
  direction: rtl;
}

[dir="rtl"] .item-info {
  text-align: right;
}

[dir="rtl"] .total-item {
  text-align: center;
}

@media (max-width: 1200px) {
  .secondary-actions {
    grid-template-columns: repeat(3, 1fr);
  }
  
  .item-card {
    margin-bottom: 6px;
  }
  
  .item-header {
    padding: 10px 12px;
  }
  
  .item-summary {
    padding: 10px 12px;
  }
}

@media (max-width: 768px) {
  .secondary-actions {
    grid-template-columns: repeat(2, 1fr);
  }
  
  .totals-container {
    flex-wrap: wrap;
    gap: 8px;
  }
  
  .total-item {
    flex: 1;
    min-width: 80px;
  }
  
  .item-name {
    font-size: 0.85rem;
  }
  
  .item-code {
    font-size: 0.75rem;
  }
  
  .qty-btn {
    width: 24px !important;
    height: 24px !important;
  }
  
  .delete-btn {
    width: 24px !important;
    height: 24px !important;
  }
}

.items-list::-webkit-scrollbar {
  width: 6px;
}

.items-list::-webkit-scrollbar-track {
  background: #f1f1f1;
  border-radius: 3px;
}

.items-list::-webkit-scrollbar-thumb {
  background: #c1c1c1;
  border-radius: 3px;
}

.items-list::-webkit-scrollbar-thumb:hover {
  background: #a1a1a1;
}

.v-enter-active,
.v-leave-active {
  transition: all 0.3s ease;
}

.v-enter-from {
  opacity: 0;
  transform: translateY(-10px);
}

.v-leave-to {
  opacity: 0;
  transform: translateY(10px);
}

@keyframes fadeIn {
  from {
    opacity: 0;
    transform: translateY(10px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.item-card {
  animation: fadeIn 0.3s ease;
}

.type-select,
.delivery-select,
.rate-field,
.date-field {
  font-size: 0.85rem;
}

.type-select :deep(.v-field),
.delivery-select :deep(.v-field),
.rate-field :deep(.v-field),
.date-field :deep(.v-field) {
  min-height: 36px !important;
}

.type-select :deep(.v-field__input),
.delivery-select :deep(.v-field__input),
.rate-field :deep(.v-field__input),
.date-field :deep(.v-field__input) {
  padding: 0 12px !important;
  font-size: 0.85rem;
}

input[type="number"] {
  -moz-appearance: textfield;
}

input[type="number"]::-webkit-outer-spin-button,
input[type="number"]::-webkit-inner-spin-button {
  -webkit-appearance: none;
  margin: 0;
}

.invoice-container.loading {
  opacity: 0.6;
  pointer-events: none;
}

.invoice-card {
  transition: all 0.3s ease;
}

.action-btn,
.pay-btn {
  transition: all 0.2s ease;
}

.action-btn:active:not(:disabled) {
  transform: scale(0.95);
}

.pay-btn:active {
  transform: scale(0.98);
}

@supports (backdrop-filter: blur(10px)) {
  .item-details {
    backdrop-filter: blur(5px);
    background: rgba(248, 249, 250, 0.9);
  }
  
  .empty-cart-content {
    backdrop-filter: blur(20px);
  }
}

@media (prefers-reduced-motion: reduce) {
  * {
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
  }
}

.item-header:focus-visible,
.qty-btn:focus-visible,
.delete-btn:focus-visible,
.action-btn:focus-visible,
.pay-btn:focus-visible {
  outline: 2px solid #1976d2;
  outline-offset: 2px;
}

@media print {
  .action-buttons,
  .qty-controls,
  .delete-btn {
    display: none !important;
  }
  
  .item-card {
    break-inside: avoid;
    page-break-inside: avoid;
  }
}
</style>