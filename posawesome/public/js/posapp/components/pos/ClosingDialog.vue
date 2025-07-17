<template>
  <v-row justify="center">
    <v-dialog v-model="closingDialog" max-width="800px">
      <v-card class="elevation-0 rounded-lg" dir="rtl">
        <v-card-title class="pa-5 bg-grey-lighten-4">
          <v-icon class="ml-3" color="primary">mdi-cash-check</v-icon>
          <span class="text-h6">{{
            __('Closing POS Shift')
          }}</span>
        </v-card-title>
        
        <v-divider></v-divider>
        
        <v-card-text class="pa-5">
          <v-container class="pa-0">
            <v-row>
              <v-col cols="12">
                <div class="text-body-1 mb-3 text-grey-darken-1">
                  {{ __('Payment Reconciliation') }}
                </div>
                
                <v-table class="border rounded" density="comfortable">
                  <thead>
                    <tr>
                      <th class="text-right bg-grey-lighten-5">{{ __('Mode of Payment') }}</th>
                      <th class="text-left bg-grey-lighten-5">{{ __('Opening Amount') }}</th>
                      <th class="text-left bg-grey-lighten-5">{{ __('Closing Amount') }}</th>
                      <th class="text-left bg-grey-lighten-5" v-if="!pos_profile.hide_expected_amount">{{ __('Expected Amount') }}</th>
                      <th class="text-left bg-grey-lighten-5" v-if="!pos_profile.hide_expected_amount">{{ __('Difference') }}</th>
                    </tr>
                  </thead>
                  <tbody>
                    <tr v-for="item in dialog_data.payment_reconciliation" :key="item.mode_of_payment">
                      <td class="text-right">{{ item.mode_of_payment }}</td>
                      <td class="text-left">
                        {{ currencySymbol(pos_profile.currency) }}
                        {{ formatCurrency(item.opening_amount) }}
                      </td>
                      <td class="text-left">
                        <v-text-field
                          v-model="item.closing_amount"
                          type="number"
                          variant="outlined"
                          density="compact"
                          hide-details
                          single-line
                          :prefix="currencySymbol(pos_profile.currency)">
                        </v-text-field>
                      </td>
                      <td class="text-left" v-if="!pos_profile.hide_expected_amount">
                        {{ currencySymbol(pos_profile.currency) }}
                        {{ formatCurrency(item.expected_amount) }}
                      </td>
                      <td class="text-left" v-if="!pos_profile.hide_expected_amount">
                        <span 
                          class="text-body-2 font-weight-medium"
                          :class="getDifferenceClass(item.expected_amount - item.closing_amount)">
                          {{ currencySymbol(pos_profile.currency) }}
                          {{ formatCurrency(item.expected_amount - item.closing_amount) }}
                        </span>
                      </td>
                    </tr>
                  </tbody>
                </v-table>
              </v-col>
            </v-row>
            
            <!-- Auto-print option -->
            <v-row class="mt-4">
              <v-col cols="12">
                <v-checkbox
                  v-model="autoPrintReceipt"
                  :label="__('Auto print closing receipt')"
                  hide-details
                  color="primary">
                </v-checkbox>
              </v-col>
            </v-row>
          </v-container>
        </v-card-text>
        
        <v-divider></v-divider>
        
        <v-card-actions class="pa-5">
          <v-spacer></v-spacer>
          <v-btn 
            variant="text"
            color="grey-darken-1"
            @click="close_dialog">
            {{ __('Close') }}
          </v-btn>
          <v-btn 
            variant="flat"
            color="primary"
            class="px-5"
            @click="submit_dialog"
            :loading="submitting">
            {{ __('Submit') }}
          </v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>
  </v-row>
</template>

<script>
import format from '../../format';
export default {
  mixins: [format],
  data: () => ({
    closingDialog: false,
    dialog_data: {},
    pos_profile: '',
    autoPrintReceipt: true, // Default to auto-print
    submitting: false,
    max25chars: (v) => v.length <= 20 || 'Input too long!',
  }),

  methods: {
    close_dialog() {
      this.closingDialog = false;
    },
    
    async submit_dialog() {
      this.submitting = true;
      
      try {
        // Emit the closing event with the data
        this.eventBus.emit('submit_closing_pos', {
          ...this.dialog_data,
          auto_print_receipt: this.autoPrintReceipt
        });
        
        // If auto-print is enabled, trigger the print after a short delay
        if (this.autoPrintReceipt) {
          // Wait for the closing to be processed
          setTimeout(() => {
            this.printClosingReceipt();
          }, 1000);
        }
        
        this.closingDialog = false;
      } catch (error) {
        console.error('Error submitting closing:', error);
        frappe.show_alert({
          message: __('Error submitting closing. Please try again.'),
          indicator: 'red'
        });
      } finally {
        this.submitting = false;
      }
    },
    
    getDifferenceClass(difference) {
      if (Math.abs(difference) < 0.01) return 'text-success';
      if (difference > 0) return 'text-warning';
      return 'text-error';
    },
    
    // Initialize closing amounts with expected amounts
    initializeClosingAmounts() {
      if (this.dialog_data.payment_reconciliation) {
        this.dialog_data.payment_reconciliation.forEach(item => {
          // Set closing amount to expected amount if not already set
          if (!item.closing_amount && item.expected_amount) {
            item.closing_amount = item.expected_amount;
          }
        });
      }
    },
    
    // Print the closing receipt
    async printClosingReceipt() {
      try {
        // Get the POS Closing Entry name from the response
        const closingEntryName = this.dialog_data.name || this.dialog_data.closing_entry_name;
        
        if (!closingEntryName) {
          // If we don't have the closing entry name yet, wait for it
          this.eventBus.once('closing_entry_created', (data) => {
            this.printClosingEntryReceipt(data.name);
          });
        } else {
          this.printClosingEntryReceipt(closingEntryName);
        }
      } catch (error) {
        console.error('Error printing receipt:', error);
        frappe.show_alert({
          message: __('Error printing receipt. You can print it manually from the POS Closing Entry.'),
          indicator: 'orange'
        });
      }
    },
    
    // Print the actual receipt
    async printClosingEntryReceipt(closingEntryName) {
      // Use frappe's print functionality
      frappe.call({
        method: 'frappe.www.printview.get_rendered_raw_commands',
        args: {
          doc: 'POS Closing Entry',
          name: closingEntryName,
          print_format: 'POS Closing Voucher', // You may need to adjust this based on your print format name
          no_letterhead: 1
        },
        callback: function(r) {
          if (r.message) {
            // For direct thermal printing
            if (window.POSPrinter && window.POSPrinter.print) {
              window.POSPrinter.print(r.message);
            } else {
              // Fallback to browser print dialog
              const printWindow = window.open('', '_blank');
              printWindow.document.write(r.message);
              printWindow.document.close();
              printWindow.print();
              printWindow.close();
            }
          }
        }
      });
    }
  },

  created: function () {
    this.eventBus.on('open_ClosingDialog', (data) => {
      this.closingDialog = true;
      this.dialog_data = data;
      
      // Initialize closing amounts with expected amounts
      this.$nextTick(() => {
        this.initializeClosingAmounts();
      });
    });
    
    this.eventBus.on('register_pos_profile', (data) => {
      this.pos_profile = data.pos_profile;
      
      // Check if auto-print is enabled in POS Profile settings
      if (data.pos_profile.auto_print_closing_receipt !== undefined) {
        this.autoPrintReceipt = data.pos_profile.auto_print_closing_receipt;
      }
    });
  },
};
</script>

<style scoped>
[dir="rtl"] .v-icon {
  transform: scaleX(-1);
}

[dir="rtl"] .ml-3 {
  margin-left: 0 !important;
  margin-right: 0.75rem !important;
}

[dir="rtl"] .mr-1 {
  margin-right: 0 !important;
  margin-left: 0.25rem !important;
}

[dir="rtl"] .text-left {
  text-align: right !important;
}

[dir="rtl"] .text-right {
  text-align: left !important;
}

.v-card {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
}

.v-table {
  border: 1px solid #e0e0e0;
}

.cursor-pointer {
  cursor: pointer;
}

.hover-bg {
  transition: background-color 0.2s ease;
}

.hover-bg:hover {
  background-color: #f5f5f5;
}

.v-field--focused {
  box-shadow: 0 0 0 2px rgba(25, 118, 210, 0.1);
}

.v-field__outline {
  color: #e0e0e0;
}

.v-btn--variant-flat {
  box-shadow: none !important;
}

.text-success {
  color: #4caf50 !important;
}

.text-warning {
  color: #ff9800 !important;
}

.text-error {
  color: #f44336 !important;
}

/* Loading state for submit button */
.v-btn--loading {
  pointer-events: none;
  opacity: 0.6;
}
</style>