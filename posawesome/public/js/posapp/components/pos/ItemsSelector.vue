<template>
  <div class="items-selector" :dir="isRTL ? 'rtl' : 'ltr'">
    <!-- UOM Selection Dialog -->
    <v-dialog v-model="uomDialog" max-width="400" persistent>
      <v-card>
        <v-card-title class="d-flex align-center justify-space-between pa-3">
          <span class="text-h6">{{ __('Select Unit') }}</span>
          <v-btn icon="mdi-close" variant="text" size="small" @click="closeUOMDialog"></v-btn>
        </v-card-title>
        
        <v-divider></v-divider>
        
        <v-card-text class="pa-3">
          <!-- Item Name -->
          <div class="text-center mb-3">
            <div class="text-subtitle-1 font-weight-medium">{{ selectedItemForUOM?.item_name }}</div>
            <div class="text-caption text-grey">{{ selectedItemForUOM?.item_code }}</div>
          </div>
          
          <!-- Quantity if from scale -->
          <div v-if="pendingQty > 1" class="text-center mb-3">
            <v-chip size="small" color="primary" variant="tonal">
              {{ __('Qty') }}: {{ formatNumber(pendingQty) }}
            </v-chip>
          </div>
          
          <!-- UOM Options -->
          <div class="uom-options">
            <v-btn
              v-for="(uom, index) in availableUOMs"
              :key="uom.uom"
              @click="selectUOM(uom)"
              :color="getUOMColor(index)"
              variant="flat"
              class="uom-btn"
              block
            >
              <div class="uom-btn-content">
                <div class="uom-name">{{ uom.uom }}</div>
                <div class="uom-price">{{ currencySymbol(selectedItemForUOM?.currency) }}{{ formatCurrency(calculateUOMPrice(uom)) }}</div>
                <div v-if="uom.conversion_factor && uom.conversion_factor !== 1" class="uom-factor">
                  {{ formatNumber(uom.conversion_factor) }}x
                </div>
              </div>
            </v-btn>
          </div>
        </v-card-text>
      </v-card>
    </v-dialog>
    <v-progress-linear v-if="loading" indeterminate color="primary" height="2" />

    <div class="controls-row">
      <v-row dense align="center" class="pa-2">
        <v-col cols="12">
          <div class="controls-container">
            <v-text-field
              v-if="pos_profile.posa_input_qty"
              v-model.number="qty"
              :label="__('Qty')"
              variant="outlined"
              density="compact"
              hide-details
              type="number"
              min="1"
              @keydown.enter="enter_event"
              class="qty-field"
            />

            <v-checkbox
              v-if="pos_profile.posa_new_line"
              v-model="new_line"
              :label="__('NL')"
              density="compact"
              hide-details
              class="new-line-check"
            />

            <v-btn-toggle v-model="items_view" mandatory density="compact" class="view-toggle">
              <v-btn value="list" size="small" icon>
                <v-icon size="small">mdi-view-list</v-icon>
              </v-btn>
              <v-btn value="card" size="small" icon>
                <v-icon size="small">mdi-view-grid</v-icon>
              </v-btn>
            </v-btn-toggle>

            <v-btn @click="show_offers" variant="outlined" size="small" color="primary" class="action-btn">
              <v-icon size="small">mdi-tag</v-icon>
              <span class="action-text">{{ offersCount }}</span>
              <v-badge v-if="appliedOffersCount > 0" :content="appliedOffersCount" color="success" floating />
            </v-btn>

            <v-btn @click="show_coupons" variant="outlined" size="small" color="secondary" class="action-btn">
              <v-icon size="small">mdi-ticket</v-icon>
              <span class="action-text">{{ couponsCount }}</span>
              <v-badge v-if="appliedCouponsCount > 0" :content="appliedCouponsCount" color="success" floating />
            </v-btn>

            <div class="search-info">
              <v-chip v-if="search" size="small" color="primary" variant="outlined" closable @click:close="clearSearch">
                "{{ search.substring(0, 10) }}{{ search.length > 10 ? '...' : '' }}"
              </v-chip>
              <div v-else class="items-count">
                <v-icon size="small" color="grey">mdi-package-variant</v-icon>
                <span>{{ filtered_items.length }} {{ __('Items') }}</span>
              </div>
            </div>
          </div>
        </v-col>
      </v-row>
    </div>

    <div class="search-field-container">
      <v-text-field
        v-model="debounce_search"
        :label="__('Search Items')"
        :placeholder="__('Search by item code, barcode, or serial number...')"
        variant="outlined"
        density="compact"
        hide-details
        clearable
        autofocus
        flat
        @keydown.enter="search_onchange"
        @keydown.esc="esc_event"
        ref="debounce_search"
        class="search-field-direct"
      >
        <template v-slot:prepend-inner>
          <v-icon size="20">mdi-magnify</v-icon>
        </template>
      </v-text-field>
    </div>

    <div class="groups-row" v-if="items_group && items_group.length > 0">
      <div class="groups-container pa-2">
        <v-btn
          v-for="group in items_group"
          :key="group"
          @click="selectItemGroup(group)"
          :variant="item_group === group ? 'flat' : 'outlined'"
          :color="item_group === group ? 'primary' : 'default'"
          size="small"
          class="group-btn"
        >
          <v-icon size="16" class="group-icon">{{ getGroupIcon(group) }}</v-icon>
          <span class="group-label">{{ getGroupDisplayName(group) }}</span>
        </v-btn>
      </div>
    </div>

    <div class="items-content">
      <div v-if="items_view === 'list'" class="list-view">
        <v-data-table
          :headers="getItemsHeaders()"
          :items="paginatedItems"
          item-key="item_code"
          density="compact"
          :items-per-page="itemsPerPage"
          hide-default-footer
          @click:row="click_item_row"
          class="items-table"
          hover
        >
          <template v-slot:item.item_code="{ item }">
            <div class="item-code-cell">{{ item.item_code }}</div>
          </template>
          <template v-slot:item.item_name="{ item }">
            <div class="d-flex align-center">
              <v-avatar size="32" class="mr-2">
                <v-img :src="item.image || '/assets/posawesome/js/posapp/components/pos/placeholder-image.png'" />
              </v-avatar>
              <div>
                <div class="item-name-wrapper">
                  <span class="item-name">{{ item.item_name }}</span>
                  <v-chip 
                    v-if="item.has_variants" 
                    size="x-small" 
                    color="primary" 
                    variant="tonal"
                    class="ml-2"
                  >
                    <v-icon size="12" start>mdi-shape-plus</v-icon>
                    {{ __('Variants') }}
                  </v-chip>
                </div>
              </div>
            </div>
          </template>
          <template v-slot:item.rate="{ item }">
            <div class="price">{{ currencySymbol(item.currency) }}{{ formatCurrency(item.rate) }}</div>
          </template>
          <template v-slot:item.actual_qty="{ item }">
            <v-chip size="small" :color="getStockColor(getItemStock(item))" variant="flat">
              {{ formatStock(item) }}
            </v-chip>
          </template>
          <template v-slot:item.stock_uom="{ item }">
            <span class="text-body-2">{{ item.stock_uom || item.uom || '-' }}</span>
          </template>
        </v-data-table>
      </div>

      <div v-if="items_view === 'card'" class="grid-view">
        <div class="cards-grid">
          <v-card
            v-for="item in paginatedItems"
            :key="item.item_code"
            @click="add_item(item)"
            class="item-card"
            hover
          >
            <div class="card-image">
              <v-img
                :src="item.image || '/assets/posawesome/js/posapp/components/pos/placeholder-image.png'"
                height="60"
                cover
              />
              <v-chip
                v-if="getItemStock(item) !== null && getItemStock(item) !== undefined"
                size="x-small"
                :color="getStockColor(getItemStock(item))"
                class="stock-badge"
              >
                {{ formatStock(item) }}
              </v-chip>
              
              <v-chip
                v-if="item.has_variants"
                size="x-small"
                color="primary"
                variant="tonal"
                class="variant-badge"
              >
                <v-icon size="14">mdi-shape-plus</v-icon>
              </v-chip>
            </div>
            <v-card-text class="pa-2">
              <div class="card-name">{{ item.item_name }}</div>
              <div class="card-price">{{ currencySymbol(item.currency) }}{{ formatCurrency(item.rate) }}</div>
              <div class="card-uom">{{ item.stock_uom }}</div>
            </v-card-text>
          </v-card>
        </div>
      </div>

      <div v-if="!loading && filtered_items.length === 0" class="empty-state">
        <v-icon size="48" color="grey">mdi-package-variant</v-icon>
        <div class="mt-2">{{ __('No items found') }}</div>
      </div>
    </div>

    <div class="pagination-bar" v-if="totalPages > 1">
      <div class="pagination-container">
        <div class="results-info">
          <span class="results-text">{{ startItem }}-{{ endItem }} {{ __('of') }} {{ filtered_items.length }}</span>
        </div>

        <div class="pagination-controls">
          <v-btn
            icon
            size="small"
            variant="text"
            :disabled="currentPage === 1"
            @click="goToFirstPage"
            class="nav-btn"
            :title="__('First Page')"
          >
            <v-icon size="18" :style="{ transform: isRTL ? 'rotate(180deg)' : 'none' }">mdi-page-first</v-icon>
          </v-btn>
          
          <v-btn
            icon
            size="small"
            variant="text"
            :disabled="currentPage === 1"
            @click="previousPage"
            class="nav-btn"
            :title="__('Previous Page')"
          >
            <v-icon size="18" :style="{ transform: isRTL ? 'rotate(180deg)' : 'none' }">mdi-chevron-left</v-icon>
          </v-btn>

          <div class="current-page-indicator">
            <span class="page-number">{{ currentPage }}</span>
            <span class="page-separator">{{ __('of') }}</span>
            <span class="total-pages">{{ totalPages }}</span>
          </div>

          <v-btn
            icon
            size="small"
            variant="text"
            :disabled="currentPage === totalPages"
            @click="nextPage"
            class="nav-btn"
            :title="__('Next Page')"
          >
            <v-icon size="18" :style="{ transform: isRTL ? 'rotate(180deg)' : 'none' }">mdi-chevron-right</v-icon>
          </v-btn>
          
          <v-btn
            icon
            size="small"
            variant="text"
            :disabled="currentPage === totalPages"
            @click="goToLastPage"
            class="nav-btn"
            :title="__('Last Page')"
          >
            <v-icon size="18" :style="{ transform: isRTL ? 'rotate(180deg)' : 'none' }">mdi-page-last</v-icon>
          </v-btn>
        </div>

        <div class="items-per-page">
          <v-select
            v-model="itemsPerPage"
            :items="[20, 50, 100, 200]"
            density="compact"
            variant="outlined"
            hide-details
            class="items-select"
            @update:model-value="currentPage = 1"
          >
            <template v-slot:selection="{ item }">
              <span class="selection-text">{{ item.value }}</span>
            </template>
          </v-select>
          <span class="per-page-label">{{ __('per page') }}</span>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import format from "../../format";
import _ from "lodash";

export default {
  mixins: [format],
  
  data: () => ({
    pos_profile: "",
    flags: {},
    items_view: "list",
    item_group: "ALL",
    loading: false,
    items_group: ["ALL"],
    items: [],
    search: "",
    first_search: "",
    offersCount: 0,
    appliedOffersCount: 0,
    couponsCount: 0,
    appliedCouponsCount: 0,
    customer_price_list: null,
    customer: null,
    new_line: false,
    qty: 1,
    isRTL: false,
    scale_barcode_data: null,
    currentPage: 1,
    itemsPerPage: 50,
    uomDialog: false,
    selectedItemForUOM: null,
    availableUOMs: [],
    pendingQty: 1,
    scannerTimeout: null,
  }),

  computed: {
    filtered_items() {
      this.search = this.get_search(this.first_search);
      let result = this.items;

      if (this.item_group !== "ALL") {
        result = result.filter(item => 
          item.item_group && item.item_group.toLowerCase().includes(this.item_group.toLowerCase())
        );
      }

      if (this.search && this.search.length >= 1) {
        const searchTerm = this.search.trim();
        const searchLower = searchTerm.toLowerCase();
        const originalSearchTerm = this.first_search ? this.first_search.trim() : '';
        
        const isScaleBarcode = (originalSearchTerm.startsWith('99') || originalSearchTerm.startsWith('91')) && 
                              (originalSearchTerm.length === 13 || originalSearchTerm.length === 12);
        
        result = result.filter(item => {
          if (item.item_barcode && item.item_barcode.length > 0) {
            const exactBarcodeMatch = item.item_barcode.some(barcode => 
              barcode.barcode === originalSearchTerm
            );
            if (exactBarcodeMatch) return true;
          }
          
          if (isScaleBarcode && this.scale_barcode_data && this.scale_barcode_data.search_keys) {
            for (const key of this.scale_barcode_data.search_keys) {
              if (item.item_code === key) return true;
              
              if (item.item_barcode && item.item_barcode.some(b => b.barcode === key)) return true;
            }
          }
          
          if (!isScaleBarcode) {
            if (item.item_code === searchTerm) return true;
            
            if (item.item_name && item.item_name.toLowerCase().includes(searchLower)) return true;
            if (item.item_code && item.item_code.toLowerCase().includes(searchLower)) return true;
          }
          
          return false;
        });
      }

      return result;
    },

    totalPages() {
      return Math.ceil(this.filtered_items.length / this.itemsPerPage);
    },

    paginatedItems() {
      const start = (this.currentPage - 1) * this.itemsPerPage;
      const end = start + this.itemsPerPage;
      return this.filtered_items.slice(start, end);
    },

    startItem() {
      if (this.filtered_items.length === 0) return 0;
      return ((this.currentPage - 1) * this.itemsPerPage) + 1;
    },

    endItem() {
      const end = this.currentPage * this.itemsPerPage;
      return end > this.filtered_items.length ? this.filtered_items.length : end;
    },

    debounce_search: {
      get() { return this.first_search; },
      set: _.debounce(function (value) { 
        this.first_search = value;
        
        if (value && value.length >= 8) {
          this.checkForBarcodeMatch(value);
        }
      }, 100),
    },
  },

  methods: {
    detectRTL() {
      const lang = frappe.boot.lang || 'en';
      this.isRTL = ['ar', 'he', 'fa', 'ur'].includes(lang);
    },

    getGroupIcon(group) {
      const icons = {
        'ALL': 'mdi-view-grid', 'Food': 'mdi-food', 'Electronics': 'mdi-cellphone',
        'Clothing': 'mdi-tshirt-crew', 'Books': 'mdi-book', 'Tools': 'mdi-tools',
        'Medicine': 'mdi-medical-bag', 'Furniture': 'mdi-chair-rolling'
      };
      const key = Object.keys(icons).find(k => group.toLowerCase().includes(k.toLowerCase()));
      return icons[key] || 'mdi-package-variant';
    },

    getGroupDisplayName(group) {
      return group === 'ALL' ? this.__('All') : (group.length > 8 ? group.substring(0, 8) + '...' : group);
    },

    getStockColor(qty) {
      const actualQty = qty || 0;
      if (actualQty <= 0) return 'error';
      if (actualQty <= 5) return 'warning';
      return 'success';
    },

    getStockClass(qty) {
      if (qty <= 0) return 'stock-out';
      if (qty <= 5) return 'stock-low';
      return 'stock-ok';
    },

    getItemStock(item) {
      return item.actual_qty || 
             item.stock_qty || 
             item.qty_available || 
             item.available_qty ||
             item.warehouse_qty ||
             item.current_qty ||
             0;
    },

    formatStock(item) {
      const qty = this.getItemStock(item);
      if (qty === 0 && item.stock_details) {
        const stockQty = item.stock_details.actual_qty || item.stock_details.qty || 0;
        return this.formatNumber(stockQty);
      }
      return this.formatNumber(qty);
    },
    
    formatNumber(value) {
      if (value === null || value === undefined) return '0';
      const num = parseFloat(value);
      if (isNaN(num)) return '0';
      return num % 1 === 0 ? num.toString() : num.toFixed(2);
    },

    hasStock(item) {
      return item.actual_qty !== undefined || 
             item.stock_qty !== undefined || 
             item.qty_available !== undefined ||
             item.available_qty !== undefined;
    },

    formatFloat(value) {
      if (value === null || value === undefined) return '0';
      const num = parseFloat(value);
      return isNaN(num) ? '0' : num.toFixed(2).replace(/\.00$/, '');
    },

    selectItemGroup(group) {
      this.item_group = group;
      if (this.pos_profile.pose_use_limit_search) {
        this.get_items();
      }
    },

    clearSearch() {
      this.first_search = '';
      this.search = '';
      this.scale_barcode_data = null;
      this.currentPage = 1;
      
      if (this.scannerTimeout) {
        clearTimeout(this.scannerTimeout);
        this.scannerTimeout = null;
      }
      
      this.$refs.debounce_search.focus();
    },

    nextPage() {
      if (this.currentPage < this.totalPages) {
        this.currentPage++;
      }
    },

    previousPage() {
      if (this.currentPage > 1) {
        this.currentPage--;
      }
    },

    goToFirstPage() {
      this.currentPage = 1;
    },

    goToLastPage() {
      this.currentPage = this.totalPages;
    },

    show_offers() { this.eventBus.emit("show_offers", "true"); },
    show_coupons() { this.eventBus.emit("show_coupons", "true"); },

    click_item_row(event, { item }) { 
      if (!item) {
        console.error('No item data in click_item_row');
        return;
      }
      this.add_item(item); 
    },

    add_item(item) {
      console.log('=== ADD ITEM CALLED ===');
      console.log('Item:', item);
      
      if (!item) {
        console.error('No item to add');
        return;
      }
      
      item = { ...item };
      
      if (item.has_variants === 1 || item.has_variants === true) {
        console.log('Item is a template with variants:', item);
        
        if (!this.eventBus) {
          console.error('EventBus not available');
          return;
        }
        
        if (!this.items || this.items.length === 0) {
          console.error('Items array not available');
          return;
        }
        
        try {
          this.eventBus.emit("open_variants_model", item, this.items);
          this.clearSearch();
        } catch (error) {
          console.error('Error opening variants dialog:', error);
        }
        return;
      }
      
      if (!item.qty) {
        if (this.scale_barcode_data && this.scale_barcode_data.qty) {
          this.pendingQty = this.scale_barcode_data.qty;
        } else {
          this.pendingQty = Math.abs(this.qty) || 1;
        }
      } else {
        this.pendingQty = item.qty;
      }
      
      console.log('Pending quantity:', this.pendingQty);
      console.log('Calling checkAndShowUOMDialog...');
      
      this.checkAndShowUOMDialog(item);
    },
    
    async checkAndShowUOMDialog(item) {
      console.log('=== CHECK AND SHOW UOM DIALOG ===');
      console.log('Item in checkAndShowUOMDialog:', item);
      
      const hasUOMData = item.uom_list || item.uoms || item.has_multiple_uoms;
      
      if (hasUOMData) {
        console.log('Item has UOM data, showing dialog...');
        this.showUOMDialog(item);
        return;
      }
      
      try {
        console.log('Fetching UOM data for item:', item.item_code);
        const response = await frappe.call({
          method: "frappe.client.get",
          args: {
            doctype: "Item",
            name: item.item_code,
            fields: ["uoms", "stock_uom", "sales_uom"]
          }
        });
        
        console.log('UOM fetch response:', response);
        
        if (response.message && response.message.uoms && response.message.uoms.length > 0) {
          item.uom_list = response.message.uoms.map(u => ({
            uom: u.uom,
            conversion_factor: u.conversion_factor
          }));
          
          console.log('Found UOMs:', item.uom_list);
          this.showUOMDialog(item);
          return;
        }
      } catch (error) {
        console.log("Could not fetch item UOM details:", error);
      }
      
      console.log('No UOM data found, adding item normally');
      item.qty = this.pendingQty;
      
      if (this.scale_barcode_data) {
        this.scale_barcode_data = null;
      }
      
      try {
        this.eventBus.emit("add_item", item);
        this.qty = 1;
        this.pendingQty = 1;
        this.clearSearch();
      } catch (error) {
        console.error('Error adding item:', error);
      }
    },
    
    showUOMDialog(item) {
      console.log('=== SHOW UOM DIALOG ===');
      console.log('Item in showUOMDialog:', item);
      
      this.selectedItemForUOM = { ...item };
      
      this.availableUOMs = item.uom_list || item.uoms || [];
      console.log('Available UOMs from item:', this.availableUOMs);
      
      if (this.availableUOMs.length === 0) {
        console.log('No UOMs found, adding item without dialog');
        item.qty = this.pendingQty;
        this.eventBus.emit("add_item", item);
        this.qty = 1;
        this.pendingQty = 1;
        if (this.scale_barcode_data) {
          this.scale_barcode_data = null;
        }
        this.clearSearch();
        return;
      }
      
      if (!this.availableUOMs.some(u => u.uom === (item.stock_uom || item.uom))) {
        this.availableUOMs.unshift({
          uom: item.stock_uom || item.uom,
          conversion_factor: 1,
          price_list_rate: item.rate
        });
      }
      
      if (this.availableUOMs.length <= 1) {
        console.log('Only one UOM available, adding item without dialog');
        item.qty = this.pendingQty;
        item.uom = this.availableUOMs[0].uom;
        item.conversion_factor = this.availableUOMs[0].conversion_factor || 1;
        this.eventBus.emit("add_item", item);
        this.qty = 1;
        this.pendingQty = 1;
        if (this.scale_barcode_data) {
          this.scale_barcode_data = null;
        }
        this.clearSearch();
        return;
      }
      
      console.log('Multiple UOMs found, showing dialog');
      console.log('Final availableUOMs:', this.availableUOMs);
      
      this.clearSearch();
      this.uomDialog = true;
    },
    
    calculateUOMPrice(uom) {
      if (!this.selectedItemForUOM) return 0;
      
      const basePrice = this.selectedItemForUOM.rate || 0;
      const conversionFactor = uom.conversion_factor || 1;
      
      if (uom.price_list_rate) {
        return uom.price_list_rate;
      }
      
      return basePrice * conversionFactor;
    },
    
    calculateTotalPrice(uom) {
      return this.calculateUOMPrice(uom) * this.pendingQty;
    },
    
    selectUOM(uom) {
      if (!this.selectedItemForUOM) return;
      
      const itemToAdd = { ...this.selectedItemForUOM };
      itemToAdd.uom = uom.uom;
      itemToAdd.conversion_factor = uom.conversion_factor || 1;
      itemToAdd.qty = this.pendingQty;
      
      itemToAdd.rate = this.calculateUOMPrice(uom);
      
      try {
        this.eventBus.emit("add_item", itemToAdd);
        this.qty = 1;
        this.pendingQty = 1;
        this.closeUOMDialog();
      } catch (error) {
        console.error('Error adding item with UOM:', error);
      }
    },
    
    closeUOMDialog() {
      this.uomDialog = false;
      this.selectedItemForUOM = null;
      this.availableUOMs = [];
      this.pendingQty = 1;
      
      if (this.scale_barcode_data) {
        this.scale_barcode_data = null;
      }
    },
    
    getUOMIcon(uom) {
      const uomLower = (uom || '').toLowerCase();
      const icons = {
        'kg': 'mdi-weight-kilogram',
        'g': 'mdi-weight-gram',
        'gram': 'mdi-weight-gram',
        'lb': 'mdi-weight-pound',
        'oz': 'mdi-weight',
        'l': 'mdi-cup',
        'liter': 'mdi-cup',
        'ml': 'mdi-cup-water',
        'pcs': 'mdi-numeric',
        'piece': 'mdi-numeric',
        'unit': 'mdi-numeric-1-box',
        'box': 'mdi-package-variant',
        'carton': 'mdi-package-variant-closed',
        'pack': 'mdi-package',
        'dozen': 'mdi-numeric-12-box',
        'm': 'mdi-ruler',
        'meter': 'mdi-ruler',
        'cm': 'mdi-ruler',
        'ft': 'mdi-ruler',
        'in': 'mdi-ruler',
        'nos': 'mdi-numeric',
        'pair': 'mdi-numeric-2-box',
        'set': 'mdi-group',
      };
      
      for (const [key, icon] of Object.entries(icons)) {
        if (uomLower.includes(key)) return icon;
      }
      
      return 'mdi-tape-measure';
    },

    onSearchInput(value) {
      if (value && value.length >= 8 && /^\d+$/.test(value)) {
        if (this.scannerTimeout) {
          clearTimeout(this.scannerTimeout);
        }
        
        this.scannerTimeout = setTimeout(() => {
          this.checkForBarcodeMatch(value);
        }, 50);
      }
    },
    
    enter_event() {
      if (this.scannerTimeout) {
        clearTimeout(this.scannerTimeout);
        this.scannerTimeout = null;
      }
      
      if (this.filtered_items.length === 1) {
        this.add_item(this.filtered_items[0]);
        this.clearSearch();
      } 
      else if (this.paginatedItems.length === 1) {
        this.add_item(this.paginatedItems[0]);
        this.clearSearch();
      }
      else if (this.paginatedItems.length > 0) {
        this.add_item(this.paginatedItems[0]);
        this.clearSearch();
      }
    },

    checkForBarcodeMatch(searchTerm) {
      if (!searchTerm) return;
      
      searchTerm = searchTerm.trim();
      console.log('Checking for barcode match:', searchTerm);
      
      if ((searchTerm.startsWith('99') || searchTerm.startsWith('91')) && 
          (searchTerm.length === 13 || searchTerm.length === 12)) {
        
        console.log('Scale barcode detected');
        
        const barcode_info = this.parse_scale_barcode(searchTerm);
        console.log('Parsed barcode info:', barcode_info);
        
        if (barcode_info.search_keys && barcode_info.search_keys.length > 0) {
          let foundItem = null;
          
          for (const key of barcode_info.search_keys) {
            foundItem = this.items.find(item => {
              if (item.item_code === key) return true;
              
              if (item.item_barcode && item.item_barcode.length > 0) {
                return item.item_barcode.some(b => b.barcode === key);
              }
              
              return false;
            });
            
            if (foundItem) {
              console.log('Found item for scale barcode:', foundItem);
              break;
            }
          }
          
          if (foundItem) {
            const itemToAdd = { ...foundItem };
            itemToAdd.qty = barcode_info.qty;
            
            console.log('Adding item with scale quantity:', itemToAdd);
            this.add_item(itemToAdd);
            return;
          } else {
            console.log('No item found for scale barcode keys:', barcode_info.search_keys);
            setTimeout(() => {
              this.clearSearch();
            }, 500);
          }
        }
        
        return;
      }
      
      const exactMatch = this.items.find(item => {
        if (item.item_code === searchTerm) return true;
        
        if (item.item_barcode && item.item_barcode.length > 0) {
          return item.item_barcode.some(barcode => barcode.barcode === searchTerm);
        }
        return false;
      });
      
      if (exactMatch) {
        console.log('Exact barcode match found:', exactMatch);
        this.add_item(exactMatch);
      } else {
        if (searchTerm.length >= 8 && /^\d+$/.test(searchTerm)) {
          setTimeout(() => {
            this.clearSearch();
          }, 500);
        }
      }
    },
    
    search_onchange() {
      const searchTerm = this.first_search ? this.first_search.trim() : '';
      
      if (searchTerm.length >= 8 && /^\d+$/.test(searchTerm)) {
        this.checkForBarcodeMatch(searchTerm);
        return;
      }
      
      if (this.pos_profile.pose_use_limit_search) {
        this.get_items();
      } else {
        if (this.filtered_items.length === 1) {
          this.add_item(this.filtered_items[0]);
          this.clearSearch();
        }
      }
    },

    esc_event() { this.clearSearch(); },

    get_items() {
      if (!this.pos_profile) return;
      this.loading = true;
      
      frappe.call({
        method: "posawesome.posawesome.api.posapp.get_items",
        args: {
          pos_profile: this.pos_profile,
          price_list: this.customer_price_list,
          item_group: this.item_group === "ALL" ? "" : this.item_group.toLowerCase(),
          customer: this.customer,
        },
        callback: (r) => {
          if (r.message) {
            console.log("Items received:", r.message);
            if (r.message.length > 0) {
              console.log("First item full data:", r.message[0]);
              console.log("All fields in first item:", Object.keys(r.message[0]));
              
              const stockFields = Object.keys(r.message[0]).filter(key => 
                key.toLowerCase().includes('qty') || 
                key.toLowerCase().includes('stock') || 
                key.toLowerCase().includes('quantity')
              );
              console.log("Potential stock fields:", stockFields);
              
              stockFields.forEach(field => {
                console.log(`${field}:`, r.message[0][field]);
              });
              
              if (r.message[0].uom_list) {
                console.log("UOM List for first item:", r.message[0].uom_list);
              }
            }
            
            this.items = r.message;
            this.eventBus.emit("set_all_items", this.items);
          }
          this.loading = false;
        },
      });
    },
    
    async fetchItemUOMs() {
      if (!this.items || this.items.length === 0) return;
      
      const hasUOMData = this.items.some(item => item.uom_list && item.uom_list.length > 0);
      if (hasUOMData) {
        console.log("Items already have UOM data");
        return;
      }
      
      try {
        if (this.pos_profile && this.pos_profile.name) {
          const response = await frappe.call({
            method: "posawesome.posawesome.api.posapp.get_item_uoms",
            args: {
              items: this.items.map(item => item.item_code)
            }
          });
          
          if (response.message) {
            this.items = this.items.map(item => {
              if (response.message[item.item_code]) {
                item.uom_list = response.message[item.item_code];
              }
              return item;
            });
          }
        }
      } catch (error) {
        console.log("Could not fetch UOM details, continuing without them");
      }
    },

    async fetchStockData() {
      if (!this.items || this.items.length === 0) return;
      
      try {
        const itemCodes = this.items.map(item => item.item_code);
        const response = await frappe.call({
          method: "erpnext.stock.dashboard.item_dashboard.get_data",
          args: {
            item_code: itemCodes,
            warehouse: this.pos_profile.warehouse || null
          }
        });
        
        if (response.message) {
          this.items = this.items.map(item => {
            const stockData = response.message.find(s => s.item_code === item.item_code);
            return {
              ...item,
              actual_qty: stockData ? stockData.actual_qty : 0
            };
          });
        }
      } catch (error) {
        console.error("Error fetching stock data:", error);
      }
    },

    get_items_groups() {
      this.items_group = ["ALL"];
      
      if (this.pos_profile?.item_groups?.length > 0) {
        this.pos_profile.item_groups.forEach(element => {
          if (element.item_group && element.item_group !== "All Item Groups") {
            if (!this.items_group.includes(element.item_group)) {
              this.items_group.push(element.item_group);
            }
          }
        });
      }
      
      if (this.items_group.length === 1 && this.items.length > 0) {
        const uniqueGroups = [...new Set(this.items.map(item => item.item_group).filter(group => group))];
        this.items_group = ["ALL", ...uniqueGroups];
      }
    },

    getItemsHeaders() {
      const headers = [
        { title: this.__("Code"), key: "item_code", width: "15%" },
        { title: this.__("Item"), key: "item_name", width: "35%" },
        { title: this.__("Price"), key: "rate", width: "15%" },
        { title: this.__("Stock"), key: "actual_qty", width: "15%" },
        { title: this.__("UOM"), key: "stock_uom", width: "20%" }
      ];
      return headers;
    },

    get_search(first_search) {
      if (!first_search) return first_search;
      
      const searchTerm = first_search.trim();
      
      if ((searchTerm.startsWith('99') || searchTerm.startsWith('91')) && 
          (searchTerm.length === 13 || searchTerm.length === 12)) {
        const barcode_info = this.parse_scale_barcode(searchTerm);
        if (barcode_info.search_keys && barcode_info.search_keys.length > 0) {
          return searchTerm;
        }
      }
      
      return searchTerm;
    },

    parse_scale_barcode(barcode) {
      const result = {
        item_code: null,
        qty: null,
        rate: null,
        search_keys: []
      };
      
      let workingBarcode = barcode;
      if (barcode.startsWith('99') && barcode.length === 13) {
        workingBarcode = barcode.slice(0, -1);
      }
      
      if (!workingBarcode.startsWith('99') && !workingBarcode.startsWith('91')) {
        return result;
      }
      
      try {
        const value_str = workingBarcode.slice(-5);
        const value = parseFloat(value_str);
        
        const is_price_scale = workingBarcode.startsWith('99');
        
        if (is_price_scale) {
          result.qty = value / 1000;
        } else {
          result.qty = value / 1000;
        }
        
        result.search_keys = [
          workingBarcode.slice(0, -5),
          workingBarcode.substring(0, 7),
          workingBarcode.substring(2, 7),
          parseInt(workingBarcode.substring(2, 7), 10).toString()
        ];
        
        result.search_keys = [...new Set(result.search_keys)];
        
        this.scale_barcode_data = result;
        
        console.log('Parsed scale barcode:', {
          original_barcode: barcode,
          working_barcode: workingBarcode,
          search_keys: result.search_keys,
          qty: result.qty
        });
        
      } catch (error) {
        console.error('Error parsing scale barcode:', error);
      }
      
      return result;
    },

    getUOMColor(index) {
      const colors = [
        'primary',
        'secondary', 
        'success',
        'warning',
        'error',
        'info'
      ];
      return colors[index % colors.length];
    },
    
    __: window.__ || ((str) => str),
  },

  watch: {
    customer() { this.get_items(); },
    new_line() { this.eventBus.emit("set_new_line", this.new_line); },
    items: {
      handler() {
        this.get_items_groups();
      },
      deep: true
    },
    filtered_items() {
      this.currentPage = 1;
    },
  },

  created() {
    this.detectRTL();
    
    this.eventBus.on("register_pos_profile", (data) => {
      this.pos_profile = data.pos_profile;
      
      console.log("POS Profile data:", this.pos_profile);
      console.log("Show stock availability:", this.pos_profile.show_stock_availability);
      console.log("Update stock:", this.pos_profile.update_stock);
      
      this.items_view = data.pos_profile.posa_default_card_view ? "card" : "list";
      this.get_items();
      this.get_items_groups();
    });

    this.eventBus.on("set_all_items", (items) => {
      console.log("Items updated from eventBus:", items);
      if (items && items.length > 0) {
        console.log("First item stock info:", {
          item_code: items[0].item_code,
          actual_qty: items[0].actual_qty,
          stock_qty: items[0].stock_qty,
          available_qty: items[0].available_qty
        });
      }
    });

    this.eventBus.on("update_offers_counters", (data) => {
      this.offersCount = data.offersCount;
      this.appliedOffersCount = data.appliedOffersCount;
    });

    this.eventBus.on("update_coupons_counters", (data) => {
      this.couponsCount = data.couponsCount;
      this.appliedCouponsCount = data.appliedCouponsCount;
    });

    this.eventBus.on("update_customer", (customer) => {
      this.customer = customer;
    });

    this.eventBus.on("update_customer_price_list", (priceList) => {
      this.customer_price_list = priceList;
    });
  },
};
</script>

<style scoped>
.items-selector {
  height: 100vh;
  display: flex;
  flex-direction: column;
  background: white;
}

.controls-row {
  background: white;
  border-bottom: 1px solid #e0e0e0;
  flex-shrink: 0;
  box-shadow: 0 1px 3px rgba(0,0,0,0.05);
}

.controls-container {
  display: flex;
  align-items: center;
  gap: 12px;
  justify-content: center;
  flex-wrap: wrap;
}

.search-field-container {
  height: 40px !important;
  flex-shrink: 0;
  background: white;
  border-bottom: 1px solid #e0e0e0;
  overflow: hidden;
}

.search-field-direct {
  margin: 0 !important;
  height: 40px !important;
}

.search-field-direct :deep(.v-input) {
  margin: 0 !important;
  height: 40px !important;
}

.search-field-direct :deep(.v-input__control) {
  height: 40px !important;
  min-height: 40px !important;
}

.search-field-direct :deep(.v-input__details) {
  display: none !important;
}

.search-field-direct :deep(.v-field) {
  background: #faf7c7  !important;
  border: none !important;
  border-radius: 0 !important;
  height: 40px !important;
}

.search-field-direct :deep(.v-field__field) {
  height: 40px !important;
}

.search-field-direct :deep(.v-field__outline) {
  display: none !important;
}

.search-field-direct :deep(.v-field:hover) {
  background: #faf7c7 !important;
}

.search-field-direct :deep(.v-field--focused) {
  background: #faf7c7 !important;
}

.search-field-direct :deep(.v-field__input) {
  font-size: 0.875rem !important;
  padding: 0 !important;
  min-height: 40px !important;
  display: flex;
  align-items: center;
}

.search-field-direct :deep(.v-field__prepend-inner) {
  padding: 0 8px 0 12px !important;
  align-items: center !important;
  display: flex !important;
  height: 40px !important;
}

.search-field-direct :deep(.v-field__append-inner) {
  padding: 0 12px 0 0 !important;
  align-items: center !important;
  display: flex !important;
  height: 40px !important;
}

.search-field-direct :deep(.v-icon) {
  color: #666;
  opacity: 0.7;
}

.search-field-direct :deep(.v-label) {
  font-size: 0.813rem !important;
  line-height: 40px !important;
  top: 0 !important;
  transform: none !important;
}

.search-field-direct :deep(.v-field-label--floating) {
  --v-field-label-scale: 0.75em;
  top: 6px !important;
  line-height: normal !important;
}

.groups-row {
  background: white;
  border-bottom: 1px solid #e0e0e0;
  flex-shrink: 0;
}

.groups-container {
  display: flex;
  gap: 8px;
  overflow-x: auto;
  scrollbar-width: none;
  -ms-overflow-style: none;
}

.groups-container::-webkit-scrollbar {
  display: none;
}

.group-btn {
  min-width: 80px !important;
  height: 36px !important;
  flex-shrink: 0;
  font-size: 0.75rem !important;
  border-radius: 6px !important;
  transition: all 0.2s ease;
}

.group-btn:hover {
  transform: translateY(-1px);
  box-shadow: 0 2px 6px rgba(0,0,0,0.1);
}

.group-icon {
  margin-right: 6px;
}

.group-label {
  font-weight: 500;
  text-transform: capitalize;
}

.qty-field {
  width: 80px;
}

.qty-field :deep(.v-field__input) {
  text-align: center;
  font-size: 0.9rem;
}

.new-line-check :deep(.v-label) {
  font-size: 0.75rem;
}

.view-toggle {
  height: 36px;
}

.view-toggle :deep(.v-btn) {
  min-width: 36px !important;
  height: 36px !important;
}

.action-btn {
  height: 36px !important;
  min-width: 80px !important;
  font-size: 0.8rem !important;
}

.action-text {
  margin-left: 6px;
  font-weight: 500;
}

.search-info {
  display: flex;
  align-items: center;
  min-height: 36px;
  gap: 8px;
}

.items-count {
  font-size: 0.85rem;
  color: #666;
  display: flex;
  align-items: center;
  gap: 6px;
  padding: 0 8px;
}

.items-content {
  flex: 1;
  overflow: hidden;
  background: white;
  margin: 0 !important;
  padding: 0 !important;
  display: flex;
  flex-direction: column;
  min-height: 0;
}

.list-view {
  height: 100%;
  overflow: hidden;
  display: flex;
  flex-direction: column;
}

.items-table {
  flex: 1;
  overflow: hidden;
  display: flex;
  flex-direction: column;
}

.items-table :deep(.v-table__wrapper) {
  flex: 1;
  overflow-y: auto;
  overflow-x: hidden;
  scrollbar-width: thin;
  scrollbar-color: #ccc #f5f5f5;
}

.items-table :deep(.v-table__wrapper::-webkit-scrollbar) {
  width: 8px;
}

.items-table :deep(.v-table__wrapper::-webkit-scrollbar-track) {
  background: #f5f5f5;
}

.items-table :deep(.v-table__wrapper::-webkit-scrollbar-thumb) {
  background-color: #ccc;
  border-radius: 4px;
}

.items-table :deep(.v-table__wrapper::-webkit-scrollbar-thumb:hover) {
  background-color: #999;
}

.items-table :deep(.v-data-table__td) {
  padding: 6px 10px !important;
  height: 48px !important;
}

.item-code-cell {
  font-family: monospace;
  font-size: 0.85rem;
  color: #666;
  font-weight: 500;
}

.item-name-wrapper {
  display: flex;
  align-items: center;
}

.item-name {
  font-weight: 700;
  font-size: 0.85rem;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  max-width: 200px;
}

.item-code {
  font-size: 0.75rem;
  color: #666;
  font-family: monospace;
}

.price {
  font-weight: 600;
  color: #1976d2;
  font-size: 0.85rem;
}

.grid-view {
  flex: 1;
  overflow-y: auto;
  overflow-x: hidden;
  padding: 8px;
  scrollbar-width: thin;
  scrollbar-color: #ccc #f5f5f5;
}

.grid-view::-webkit-scrollbar {
  width: 8px;
}

.grid-view::-webkit-scrollbar-track {
  background: #f5f5f5;
}

.grid-view::-webkit-scrollbar-thumb {
  background-color: #ccc;
  border-radius: 4px;
}

.grid-view::-webkit-scrollbar-thumb:hover {
  background-color: #999;
}

.cards-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(120px, 1fr));
  gap: 8px;
}

.item-card {
  cursor: pointer;
  transition: transform 0.2s;
  border-radius: 6px !important;
  min-height: 130px;
}

.item-card:hover {
  transform: translateY(-2px);
}

.card-image {
  position: relative;
}

.stock-badge {
  position: absolute;
  top: 4px;
  right: 4px;
  font-size: 0.65rem !important;
  height: 18px !important;
}

.variant-badge {
  position: absolute;
  top: 4px;
  left: 4px;
  font-size: 0.65rem !important;
  height: 18px !important;
  min-width: 24px !important;
  padding: 0 4px !important;
}

.card-name {
  font-weight: 700;
  font-size: 0.75rem;
  margin-bottom: 4px;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.card-price {
  font-weight: 900;
  color: #1976d2;
  font-size: 0.8rem;
  margin-bottom: 2px;
}

.card-uom {
  font-size: 0.65rem;
  color: #666;
  font-weight: 500;
}

.stock-ok { color: #4caf50; }
.stock-low { color: #ff9800; }
.stock-out { color: #f44336; }

.empty-state {
  flex: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  color: #666;
}

.pagination-bar {
  background: linear-gradient(135deg, #f8f9fa 0%, #e9ecef 100%);
  border-top: 1px solid #dee2e6;
  flex-shrink: 0;
  padding: 8px 16px;
  box-shadow: 0 -2px 8px rgba(0,0,0,0.05);
}

.pagination-container {
  display: flex;
  align-items: center;
  justify-content: space-between;
  max-width: 100%;
  gap: 16px;
}

.results-info {
  display: flex;
  align-items: center;
  min-width: fit-content;
}

.results-text {
  font-size: 0.813rem;
  color: #495057;
  font-weight: 500;
  white-space: nowrap;
}

.pagination-controls {
  display: flex;
  align-items: center;
  gap: 4px;
  flex: 1;
  justify-content: center;
}

.nav-btn {
  width: 36px !important;
  height: 36px !important;
  border-radius: 8px !important;
  color: #6c757d !important;
  transition: all 0.2s cubic-bezier(0.4, 0, 0.2, 1) !important;
  background: rgba(255, 255, 255, 0.7) !important;
  border: 1px solid rgba(0, 0, 0, 0.1) !important;
}

.nav-btn:hover:not(:disabled) {
  color: #1976d2 !important;
  background: rgba(25, 118, 210, 0.04) !important;
  border-color: rgba(25, 118, 210, 0.2) !important;
  transform: translateY(-1px);
  box-shadow: 0 4px 12px rgba(25, 118, 210, 0.15);
}

.nav-btn:disabled {
  color: #adb5bd !important;
  background: rgba(248, 249, 250, 0.5) !important;
  border-color: transparent !important;
  cursor: not-allowed;
}

.nav-btn :deep(.v-btn__content) {
  opacity: 1;
}

.current-page-indicator {
  display: flex;
  align-items: center;
  gap: 6px;
  background: rgba(25, 118, 210, 0.08);
  padding: 6px 12px;
  border-radius: 12px;
  border: 1px solid rgba(25, 118, 210, 0.2);
  margin: 0 8px;
}

.page-number {
  font-size: 0.875rem;
  font-weight: 700;
  color: #1976d2;
  min-width: 20px;
  text-align: center;
}

.page-separator {
  font-size: 0.75rem;
  color: #6c757d;
  font-weight: 500;
}

.total-pages {
  font-size: 0.875rem;
  font-weight: 600;
  color: #495057;
  min-width: 20px;
  text-align: center;
}

.items-per-page {
  display: flex;
  align-items: center;
  gap: 8px;
  min-width: fit-content;
}

.items-select {
  min-width: 65px;
  max-width: 80px;
}

.items-select :deep(.v-field) {
  background: rgba(255, 255, 255, 0.9) !important;
  border-radius: 6px !important;
  min-height: 32px !important;
  font-size: 0.813rem !important;
}

.items-select :deep(.v-field__input) {
  font-size: 0.813rem !important;
  font-weight: 600 !important;
  color: #495057 !important;
  text-align: center !important;
  padding: 0 4px !important;
  min-height: 32px !important;
}

.items-select :deep(.v-field__append-inner) {
  padding: 0 6px !important;
}

.items-select :deep(.v-icon) {
  font-size: 16px !important;
  color: #6c757d !important;
}

.selection-text {
  font-weight: 600;
  color: #495057;
}

.per-page-label {
  font-size: 0.75rem;
  color: #6c757d;
  font-weight: 500;
  white-space: nowrap;
}

.pagination-controls .nav-btn,
.current-page-indicator,
.items-select {
  transition: all 0.2s cubic-bezier(0.4, 0, 0.2, 1);
}

[dir="rtl"] .price,
[dir="rtl"] .card-price {
  direction: ltr;
  text-align: right;
}

[dir="rtl"] .stock-badge {
  right: auto;
  left: 4px;
}

[dir="rtl"] .group-icon {
  margin-right: 0;
  margin-left: 6px;
}

[dir="rtl"] .action-text {
  margin-left: 0;
  margin-right: 6px;
}

[dir="rtl"] .search-field-direct :deep(.v-field__prepend-inner) {
  padding-right: 0;
  padding-left: 12px;
}

[dir="rtl"] .pagination-container {
  direction: rtl;
}

[dir="rtl"] .results-info {
  order: 3;
}

[dir="rtl"] .pagination-controls {
  order: 2;
}

[dir="rtl"] .items-per-page {
  order: 1;
}

@media (max-width: 1200px) {
  .controls-container {
    gap: 8px;
  }
  
  .action-btn {
    min-width: 70px !important;
    font-size: 0.75rem !important;
  }
  
  .action-text {
    font-size: 0.75rem;
  }
  
  .group-btn {
    min-width: 70px !important;
    height: 34px !important;
  }
  
  .pagination-container {
    gap: 12px;
  }
  
  .nav-btn {
    width: 32px !important;
    height: 32px !important;
  }
}

@media (max-width: 768px) {
  .cards-grid {
    grid-template-columns: repeat(auto-fill, minmax(100px, 1fr));
    gap: 6px;
  }
  
  .action-text {
    display: none;
  }
  
  .action-btn {
    min-width: 40px !important;
  }
  
  .group-btn {
    min-width: 60px !important;
    height: 32px !important;
  }
  
  .group-label {
    font-size: 0.65rem;
  }
  
  .pagination-container {
    flex-direction: column;
    gap: 8px;
    align-items: center;
  }
  
  .results-info {
    order: 1;
  }
  
  .pagination-controls {
    order: 2;
    gap: 2px;
  }
  
  .items-per-page {
    order: 3;
  }
  
  .nav-btn {
    width: 28px !important;
    height: 28px !important;
  }
  
  .current-page-indicator {
    padding: 4px 8px;
    margin: 0 4px;
  }
  
  .page-number,
  .total-pages {
    font-size: 0.8rem;
  }
  
  .page-separator {
    font-size: 0.7rem;
  }
}

@media (max-width: 480px) {
  .group-label {
    display: none;
  }
  
  .group-btn {
    min-width: 40px !important;
    height: 30px !important;
  }
  
  .group-icon {
    margin: 0;
  }
  
  .action-btn {
    min-width: 36px !important;
    height: 32px !important;
  }
  
  .controls-container {
    gap: 6px;
    padding: 0 8px;
  }
  
  .per-page-label {
    display: none;
  }
  
  .results-text {
    font-size: 0.75rem;
  }
  
  .pagination-bar {
    padding: 6px 8px;
  }
}

@keyframes slideIn {
  from {
    opacity: 0;
    transform: translateY(10px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.pagination-bar {
  animation: slideIn 0.3s ease-out;
}

.nav-btn:active {
  transform: translateY(0) !important;
  box-shadow: 0 2px 6px rgba(25, 118, 210, 0.2) !important;
}

.current-page-indicator {
  position: relative;
  overflow: hidden;
}

.current-page-indicator::before {
  content: '';
  position: absolute;
  top: 0;
  left: -100%;
  width: 100%;
  height: 100%;
  background: linear-gradient(90deg, transparent, rgba(255,255,255,0.4), transparent);
  animation: shimmer 2s infinite;
}

@keyframes shimmer {
  0% {
    left: -100%;
  }
  100% {
    left: 100%;
  }
}

.uom-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(150px, 1fr));
  gap: 12px;
}

.uom-btn {
  text-transform: none !important;
  flex-direction: column !important;
  padding: 12px !important;
}

.uom-btn :deep(.v-btn__content) {
  flex-direction: column !important;
  width: 100%;
}

.v-dialog {
  z-index: 2000 !important;
}

.uom-options {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.uom-btn {
  height: auto !important;
  padding: 12px 16px !important;
  text-transform: none !important;
  letter-spacing: normal !important;
}

.uom-btn-content {
  display: flex;
  align-items: center;
  justify-content: space-between;
  width: 100%;
  gap: 12px;
}

.uom-name {
  font-size: 1rem;
  font-weight: 500;
}

.uom-price {
  font-size: 1.1rem;
  font-weight: 700;
  margin-left: auto;
}

.uom-factor {
  font-size: 0.85rem;
  opacity: 0.9;
  font-weight: 500;
  padding: 2px 8px;
  background: rgba(255,255,255,0.2);
  border-radius: 12px;
}

.v-btn--variant-flat {
  color: white !important;
}

.v-btn--variant-flat .uom-btn-content > * {
  color: white !important;
}

@media (max-width: 400px) {
  .v-dialog {
    margin: 16px !important;
  }
}

[dir="rtl"] .uom-btn-content {
  flex-direction: row-reverse;
}

[dir="rtl"] .uom-price {
  margin-left: 0;
  margin-right: auto;
}
</style>