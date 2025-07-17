<template>
  <nav>
    <div 
      class="navbar-container"
      :class="{ 
        'navbar-hidden': !navbarVisible, 
        'navbar-locked': isNavbarLocked 
      }"
      @mouseenter="onNavbarEnter"
      @mouseleave="onNavbarLeave"
    >
      <v-toolbar 
        height="56" 
        class="navbar-toolbar"
        color="white"
        elevation="2"
      >
        <v-toolbar-title @click="go_desk" style="cursor: pointer" class="text-uppercase text-primary mx-3">
          <span class="font-weight-light">HIGHSPEED</span>
          <span class="font-weight-bold ml-1">IT</span>
        </v-toolbar-title>

        <v-spacer></v-spacer>

        <div class="d-flex align-center">
          <v-btn 
            :variant="currentPage === 'POS' ? 'tonal' : 'text'"
            color="primary"
            size="small"
            @click="changePage('POS')"
            class="mx-1"
          >
            <v-icon size="20" :start="!isRTL" :end="isRTL">mdi-network-pos</v-icon>
            <span>{{ __('POS') }}</span>
          </v-btn>

          <v-btn 
            v-if="pos_profile.posa_use_pos_awesome_payments"
            :variant="currentPage === 'Payments' ? 'tonal' : 'text'"
            color="primary"
            size="small"
            @click="changePage('Payments')"
            class="mx-1"
          >
            <v-icon size="20" :start="!isRTL" :end="isRTL">mdi-cash-register</v-icon>
            <span>{{ __('Payments') }}</span>
          </v-btn>

          <v-chip variant="outlined" color="primary" size="small" class="mx-2">
            <v-icon size="16" :start="!isRTL" :end="isRTL">mdi-store</v-icon>
            <span>{{ pos_profile.name }}</span>
          </v-chip>
        </div>

        <v-spacer></v-spacer>

        <div class="d-flex align-center" :class="{ 'flex-row-reverse': isRTL }">
          <v-btn 
            icon 
            @click="showInvoicesDialog = true"
            variant="text" 
            color="primary"
            size="small"
            :title="__('Recent Invoices')"
            class="mx-1"
          >
            <v-badge 
              v-if="lastInvoices.length > 0" 
              :content="lastInvoices.length"
              color="error"
              dot
            >
              <v-icon>mdi-receipt-text</v-icon>
            </v-badge>
            <v-icon v-else>mdi-receipt-text</v-icon>
          </v-btn>

          <v-btn 
            v-if="!pos_profile.posa_hide_closing_shift"
            icon 
            @click="close_shift_dialog"
            variant="text" 
            color="primary"
            size="small"
            :title="__('Close Shift')"
            class="mx-1"
          >
            <v-icon>mdi-content-save-move-outline</v-icon>
          </v-btn>

          <v-btn 
            v-if="pos_profile.posa_allow_print_last_invoice && last_invoice"
            icon 
            @click="print_last_invoice"
            variant="text" 
            color="primary"
            size="small"
            :title="__('Print Last Invoice')"
            class="mx-1"
          >
            <v-icon>mdi-printer</v-icon>
          </v-btn>

          <v-divider vertical class="mx-2" style="height: 24px;"></v-divider>

          <v-btn 
            icon 
            @click.stop="toggleNavbarLock" 
            variant="text" 
            color="primary"
            size="small"
            :title="isNavbarLocked ? __('Unpin Navbar') : __('Pin Navbar')"
            class="mx-1"
          >
            <v-icon>{{ isNavbarLocked ? 'mdi-pin' : 'mdi-pin-outline' }}</v-icon>
          </v-btn>

          <v-btn 
            icon 
            @click.stop="toggleFullScreen" 
            variant="text" 
            color="primary"
            size="small"
            :title="isFullscreen ? __('Exit Fullscreen') : __('Fullscreen')"
            class="mx-1"
          >
            <v-icon>{{ isFullscreen ? 'mdi-fullscreen-exit' : 'mdi-fullscreen' }}</v-icon>
          </v-btn>

          <v-btn 
            icon 
            @click="logOut"
            variant="text" 
            color="primary"
            size="small"
            :title="__('Logout')"
            class="mx-1"
          >
            <v-icon>mdi-logout</v-icon>
          </v-btn>
        </div>
      </v-toolbar>
    </div>

    <div 
      v-if="!navbarVisible && !isNavbarLocked"
      class="navbar-hover-zone"
      @mouseenter="showNavbar"
    ></div>

    <v-dialog v-model="showInvoicesDialog" max-width="1000" scrollable>
      <v-card class="invoice-dialog">
        <v-card-title class="invoice-header d-flex align-center">
          <span class="text-h6">{{ __('Recent Invoices') }}</span>
          <v-spacer></v-spacer>
          <v-btn icon variant="text" @click="showInvoicesDialog = false">
            <v-icon>mdi-close</v-icon>
          </v-btn>
        </v-card-title>
        
        <v-divider></v-divider>
        
        <v-card-text class="pa-0">
          <v-table class="invoice-table" fixed-header height="450">
            <thead>
              <tr>
                <th class="text-center" width="150">{{ __('Actions') }}</th>
                <th :class="isRTL ? 'text-right' : 'text-left'" width="200">{{ __('Amount') }}</th>
                <th :class="isRTL ? 'text-right' : 'text-left'">{{ __('Date') }}</th>
                <th :class="isRTL ? 'text-right' : 'text-left'">{{ __('Customer') }}</th>
                <th :class="isRTL ? 'text-right' : 'text-left'">{{ __('Invoice') }}</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="invoice in lastInvoices" :key="invoice.name" class="invoice-row">
                <td class="text-center">
                  <v-btn 
                    icon 
                    size="small" 
                    @click="printInvoiceDirect(invoice.name)"
                    color="primary"
                    variant="tonal"
                    class="ma-1"
                  >
                    <v-icon size="20">mdi-printer</v-icon>
                  </v-btn>
                  <v-btn 
                    icon 
                    size="small" 
                    @click="viewInvoice(invoice.name)"
                    color="grey"
                    variant="tonal"
                    class="ma-1"
                  >
                    <v-icon size="20">mdi-eye</v-icon>
                  </v-btn>
                </td>
                <td :class="isRTL ? 'text-right' : 'text-left'">
                  <span class="font-weight-medium">{{ currencySymbol(invoice.currency) }}{{ formatCurrency(invoice.grand_total) }}</span>
                </td>
                <td :class="isRTL ? 'text-right' : 'text-left'">{{ formatDate(invoice.posting_date) }}</td>
                <td :class="isRTL ? 'text-right' : 'text-left'">{{ invoice.customer_name || __('Walk-in Customer') }}</td>
                <td :class="isRTL ? 'text-right' : 'text-left'">{{ invoice.name }}</td>
              </tr>
            </tbody>
          </v-table>
          
          <div v-if="lastInvoices.length === 0" class="empty-state text-center pa-8">
            <v-icon size="64" color="grey-lighten-2">mdi-receipt-text-remove</v-icon>
            <p class="text-h6 mt-4">{{ __('No invoices found') }}</p>
            <p class="text-body-2 text-grey">{{ __('Your recent invoices will appear here') }}</p>
          </div>
        </v-card-text>
      </v-card>
    </v-dialog>

    <v-snackbar 
      v-model="snack" 
      :timeout="5000" 
      :color="snackColor" 
      :location="isRTL ? 'top left' : 'top right'"
    >
      {{ snackText }}
      <template v-slot:actions>
        <v-btn variant="text" @click="snack = false">
          <v-icon>mdi-close</v-icon>
        </v-btn>
      </template>
    </v-snackbar>

    <v-dialog v-model="freeze" persistent max-width="320">
      <v-card>
        <v-card-title>{{ freezeTitle }}</v-card-title>
        <v-card-text>{{ freezeMsg }}</v-card-text>
        <v-card-actions>
          <v-spacer></v-spacer>
          <v-progress-circular indeterminate color="primary" size="24"></v-progress-circular>
        </v-card-actions>
      </v-card>
    </v-dialog>
  </nav>
</template>

<script>
export default {
  data() {
    return {
      currentPage: 'POS',
      snack: false,
      snackColor: '',
      snackText: '',
      company: 'HIGHSPEED IT',
      company_img: '/assets/erpnext/images/erpnext-logo.svg',
      pos_profile: '',
      freeze: false,
      freezeTitle: '',
      freezeMsg: '',
      last_invoice: '',
      lastInvoices: [],
      showInvoicesDialog: false,
      isFullscreen: false,
      isNavbarLocked: false,
      navbarVisible: false,
      hideTimer: null,
      isRTL: false,
    };
  },
  
  methods: {
    showNavbar() {
      this.navbarVisible = true;
      this.clearHideTimer();
    },
    
    hideNavbar() {
      if (!this.isNavbarLocked) {
        this.navbarVisible = false;
      }
    },
    
    onNavbarEnter() {
      this.showNavbar();
    },
    
    onNavbarLeave() {
      if (!this.isNavbarLocked) {
        this.clearHideTimer();
        this.hideTimer = setTimeout(() => {
          this.hideNavbar();
        }, 500);
      }
    },
    
    clearHideTimer() {
      if (this.hideTimer) {
        clearTimeout(this.hideTimer);
        this.hideTimer = null;
      }
    },
    
    changePage(key) {
      this.currentPage = key;
      this.$emit('changePage', key);
    },
    
    go_desk() {
      frappe.set_route('/');
      location.reload();
    },
    
    close_shift_dialog() {
      this.eventBus.emit('open_closing_dialog');
    },
    
    show_message(data) {
      this.snack = true;
      this.snackColor = data.color;
      this.snackText = data.title;
    },
    
    logOut() {
      frappe.call({
        method: 'logout',
        callback: function (r) {
          if (!r.exc) {
            frappe.set_route('/login');
            location.reload();
          }
        },
      });
    },
    
    print_last_invoice() {
      if (!this.last_invoice) return;
      this.printInvoiceDirect(this.last_invoice);
    },
    
    printInvoiceDirect(invoiceName) {
      if (!invoiceName) return;
      
      const print_format = this.pos_profile.print_format_for_online || this.pos_profile.print_format;
      const letter_head = this.pos_profile.letter_head || 0;
      
      const printFrame = document.createElement('iframe');
      printFrame.style.display = 'none';
      printFrame.src = frappe.urllib.get_base_url() +
        '/printview?doctype=Sales%20Invoice&name=' +
        invoiceName +
        '&format=' +
        print_format +
        '&no_letterhead=' +
        letter_head;
      
      document.body.appendChild(printFrame);
      
      printFrame.onload = function() {
        setTimeout(() => {
          printFrame.contentWindow.print();
          setTimeout(() => {
            document.body.removeChild(printFrame);
          }, 100);
        }, 250);
      };
    },
    
    loadLastInvoices() {
      frappe.call({
        method: 'frappe.client.get_list',
        args: {
          doctype: 'Sales Invoice',
          fields: ['name', 'customer_name', 'grand_total', 'posting_date', 'posting_time', 'currency'],
          filters: {
            docstatus: 1,
            pos_profile: this.pos_profile.name
          },
          order_by: 'creation desc',
          limit: 10
        },
        callback: (r) => {
          if (r.message) {
            this.lastInvoices = r.message;
          }
        }
      });
    },
    
    formatDate(date) {
      if (!date) return '';
      return new Date(date).toLocaleDateString();
    },
    
    viewInvoice(invoiceName) {
      const url = frappe.urllib.get_base_url() + '/app/sales-invoice/' + invoiceName;
      window.open(url, '_blank');
    },
    
    currencySymbol(currency) {
      const symbols = {
        'USD': '$',
        'EUR': '€',
        'GBP': '£',
        'SAR': 'ر.س',
        'AED': 'د.إ',
        'EGP': 'ج.م',
        'INR': '₹'
      };
      return symbols[currency] || currency + ' ';
    },
    
    formatCurrency(value) {
      if (!value) return '0.00';
      return parseFloat(value).toLocaleString(undefined, {
        minimumFractionDigits: 2,
        maximumFractionDigits: 2
      });
    },
    
    toggleFullScreen() {
      if (!document.fullscreenElement) {
        document.documentElement.requestFullscreen().then(() => {
          this.isFullscreen = true;
        }).catch((err) => {
          console.warn(`Error attempting to enable full-screen mode: ${err.message}`);
        });
      } else {
        document.exitFullscreen().then(() => {
          this.isFullscreen = false;
        });
      }
    },
    
    toggleNavbarLock() {
      this.isNavbarLocked = !this.isNavbarLocked;
      localStorage.setItem('navbar_locked', this.isNavbarLocked.toString());
      
      if (this.isNavbarLocked) {
        this.navbarVisible = true;
        document.body.style.paddingTop = '56px';
      } else {
        document.body.style.paddingTop = '0';
        setTimeout(() => {
          this.navbarVisible = false;
        }, 300);
      }
    },
    
    loadNavbarState() {
      const savedState = localStorage.getItem('navbar_locked');
      if (savedState !== null) {
        this.isNavbarLocked = savedState === 'true';
        this.navbarVisible = this.isNavbarLocked;
        
        if (this.isNavbarLocked) {
          document.body.style.paddingTop = '56px';
        }
      }
    },
    
    detectRTL() {
      const htmlDir = document.documentElement.dir;
      const lang = document.documentElement.lang;
      this.isRTL = htmlDir === 'rtl' || ['ar', 'he', 'fa', 'ur'].includes(lang.substring(0, 2));
    },

    __: window.__ || ((str) => str),
  },
  
  created() {
    this.loadNavbarState();
    this.detectRTL();
    
    this.$nextTick(() => {
      this.eventBus.on('show_message', (data) => this.show_message(data));
      this.eventBus.on('set_company', (data) => {
        this.company = data.name;
        this.company_img = data.company_logo || this.company_img;
      });
      this.eventBus.on('register_pos_profile', (data) => {
        this.pos_profile = data.pos_profile;
        this.loadLastInvoices();
      });
      this.eventBus.on('set_last_invoice', (data) => {
        this.last_invoice = data;
        this.loadLastInvoices();
      });
      this.eventBus.on('freeze', (data) => {
        this.freeze = true;
        this.freezeTitle = data.title;
        this.freezeMsg = data.msg;
      });
      this.eventBus.on('unfreeze', () => {
        this.freeze = false;
        this.freezeTitle = '';
        this.freezeMsg = '';
      });
    });
  },
  
  mounted() {
    const observer = new MutationObserver(() => {
      this.detectRTL();
    });
    
    observer.observe(document.documentElement, {
      attributes: true,
      attributeFilter: ['dir', 'lang']
    });
  },
  
  beforeUnmount() {
    this.clearHideTimer();
    document.body.style.paddingTop = '0';
  },
};
</script>

<style scoped>
.navbar-container {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  z-index: 50;
  transition: transform 0.3s ease, opacity 0.3s ease;
  transform: translateY(0);
  opacity: 1;
}

.navbar-container.navbar-hidden {
  transform: translateY(-85%);
  opacity: 0.9;
}

.navbar-container.navbar-locked {
  position: relative;
  z-index: auto;
  transform: translateY(0) !important;
  opacity: 1 !important;
}

.navbar-toolbar {
  backdrop-filter: blur(10px);
  background: rgba(255, 255, 255, 0.95) !important;
  border-bottom: 1px solid rgba(0, 0, 0, 0.08);
}

.navbar-hover-zone {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  height: 20px;
  z-index: 49;
  background: transparent;
  cursor: s-resize;
}

.invoice-dialog {
  border-radius: 8px !important;
}

.invoice-header {
  background-color: #f5f5f5;
  padding: 16px 24px;
}

.invoice-table {
  background: white;
}

.invoice-table thead {
  background-color: #fafafa;
}

.invoice-table th {
  font-weight: 600 !important;
  color: #666 !important;
  font-size: 0.875rem !important;
  padding: 12px 16px !important;
  border-bottom: 2px solid #e0e0e0 !important;
}

.invoice-row {
  transition: background-color 0.2s ease;
}

.invoice-row:hover {
  background-color: #f9f9f9;
}

.invoice-row td {
  padding: 10px 16px !important;
  border-bottom: 1px solid #f0f0f0 !important;
  font-size: 0.875rem;
}

.empty-state {
  min-height: 300px;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  background-color: #fafafa;
}

body {
  transition: padding-top 0.3s ease;
}

[dir="rtl"] .invoice-table {
  direction: rtl;
}

[dir="rtl"] .invoice-header {
  direction: rtl;
}

@media (max-width: 768px) {
  .v-toolbar-title {
    font-size: 0.9rem;
  }
  
  .v-chip {
    display: none !important;
  }
  
  .navbar-toolbar .v-btn span {
    display: none;
  }
  
  .navbar-toolbar .v-btn {
    min-width: 40px !important;
    width: 40px !important;
  }
  
  .invoice-dialog {
    margin: 8px;
  }
}
</style>