<template>
  <v-row justify="center">
    <v-dialog v-model="draftsDialog" max-width="1000px" transition="dialog-transition" scrollable>
      <v-card elevation="0" class="dialog-card">
        <v-card-title class="dialog-header">
          <div class="header-content">
            <v-icon size="28" color="primary" class="mr-3">mdi-file-document-multiple</v-icon>
            <div>
              <h3 class="dialog-title">{{ __('Load Sales Invoice') }}</h3>
              <p class="dialog-subtitle">{{ __('Load previously saved invoices') }}</p>
            </div>
          </div>
        </v-card-title>

        <v-divider></v-divider>

        <v-card-text class="pa-0">
          <v-container>
            <v-row no-gutters>
              <v-col cols="12" class="pa-1">
                <v-data-table 
                  :headers="headers" 
                  :items="dialog_data" 
                  item-value="name" 
                  class="elevation-1 invoice-table" 
                  show-select
                  v-model="selected" 
                  select-strategy="single" 
                  return-object
                  hover
                >
                  <template v-slot:item.customer_name="{ item }">
                    <div class="customer-cell">
                      <v-avatar size="32" color="primary" class="mr-2">
                        <span class="text-caption white--text">
                          {{ getInitials(item.customer_name) }}
                        </span>
                      </v-avatar>
                      <div>
                        <div class="customer-name">{{ item.customer_name }}</div>
                        <div class="invoice-number">{{ item.name }}</div>
                      </div>
                    </div>
                  </template>

                  <template v-slot:item.posting_date="{ item }">
                    <div class="date-cell">
                      <v-icon size="16" class="mr-1">mdi-calendar</v-icon>
                      {{ item.posting_date }}
                    </div>
                  </template>

                  <template v-slot:item.posting_time="{ item }">
                    <div class="time-cell">
                      <v-icon size="16" class="mr-1">mdi-clock-outline</v-icon>
                      {{ item.posting_time.split('.')[0] }}
                    </div>
                  </template>

                  <template v-slot:item.grand_total="{ item }">
                    <div class="amount-cell">
                      <span class="currency">{{ currencySymbol(item.currency) }}</span>
                      <span class="amount">{{ formatCurrency(item.grand_total) }}</span>
                    </div>
                  </template>
                </v-data-table>
              </v-col>
            </v-row>
          </v-container>
        </v-card-text>

        <v-divider></v-divider>

        <v-card-actions class="dialog-actions">
          <v-spacer></v-spacer>
          <v-btn 
            variant="text" 
            color="error" 
            @click="close_dialog"
            class="action-btn"
          >
            <v-icon size="18" class="mr-2">mdi-close</v-icon>
            {{ __('Close') }}
          </v-btn>
          <v-btn 
            variant="flat" 
            color="success" 
            @click="submit_dialog"
            class="action-btn primary-btn"
            elevation="0"
          >
            <v-icon size="18" class="mr-2">mdi-file-import</v-icon>
            {{ __('Load Sale') }}
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
    draftsDialog: false,
    singleSelect: true,
    selected: [],
    dialog_data: {},
    headers: [
      {
        title: __('Customer'),
        value: 'customer_name',
        align: 'start',
        sortable: true,
      },
      {
        title: __('Date'),
        align: 'start',
        sortable: true,
        value: 'posting_date',
      },
      {
        title: __('Time'),
        align: 'start',
        sortable: true,
        value: 'posting_time',
      },
      {
        title: __('Invoice'),
        value: 'name',
        align: 'start',
        sortable: true,
      },
      {
        title: __('Amount'),
        value: 'grand_total',
        align: 'end',
        sortable: false,
      },
    ],
  }),
  watch: {},
  methods: {
    close_dialog() {
      this.draftsDialog = false;
    },

    submit_dialog() {
      if (this.selected.length > 0) {
        this.eventBus.emit('load_invoice', this.selected[0]);
        this.draftsDialog = false;
      }
      else {
        this.eventBus.emit("show_message", {
          title: __('Select an invoice to load'),
          color: "error",
        });
      }
    },

    getInitials(name) {
      if (!name) return '?';
      return name
        .split(' ')
        .map(word => word[0])
        .join('')
        .toUpperCase()
        .substring(0, 2);
    },

    __: window.__ || ((str) => str),
  },
  created: function () {
    this.eventBus.on('open_drafts', (data) => {
      this.draftsDialog = true;
      this.dialog_data = data;
    });
  },
  beforeUnmount() {
    this.eventBus.off('open_drafts');
  },
};
</script>

<style scoped>
.dialog-card {
  border-radius: 16px !important;
  overflow: hidden;
}

.dialog-header {
  background: linear-gradient(135deg, #f8f9fa 0%, #ffffff 100%);
  padding: 24px;
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
}

.header-content {
  display: flex;
  align-items: flex-start;
}

.dialog-title {
  font-size: 1.25rem;
  font-weight: 600;
  color: #1e293b;
  margin: 0;
  line-height: 1.2;
}

.dialog-subtitle {
  font-size: 0.875rem;
  color: #64748b;
  margin: 4px 0 0 0;
  font-weight: 400;
}

.invoice-table {
  background: transparent !important;
  border-radius: 0 !important;
}

.invoice-table :deep(.v-table__wrapper) {
  max-height: 450px;
  overflow-y: auto;
}

.invoice-table :deep(thead) {
  background: #f8fafc !important;
}

.invoice-table :deep(th) {
  font-weight: 600 !important;
  color: #475569 !important;
  font-size: 0.875rem !important;
  padding: 16px !important;
  border-bottom: 1px solid #e2e8f0 !important;
}

.invoice-table :deep(tbody tr) {
  transition: all 0.2s ease;
}

.invoice-table :deep(tbody tr:hover) {
  background: #f8fafc !important;
}

.invoice-table :deep(tbody tr.v-data-table__selected) {
  background: #eff6ff !important;
}

.customer-cell {
  display: flex;
  align-items: center;
  padding: 8px 0;
}

.customer-name {
  font-weight: 500;
  color: #1e293b;
  font-size: 0.875rem;
}

.invoice-number {
  font-size: 0.75rem;
  color: #64748b;
  font-family: monospace;
}

.date-cell,
.time-cell {
  display: flex;
  align-items: center;
  color: #475569;
  font-size: 0.875rem;
}

.date-cell .v-icon,
.time-cell .v-icon {
  color: #94a3b8;
}

.amount-cell {
  display: flex;
  align-items: baseline;
  justify-content: flex-end;
  gap: 4px;
}

.currency {
  font-size: 0.875rem;
  color: #64748b;
  font-weight: 500;
}

.amount {
  font-size: 1rem;
  font-weight: 600;
  color: #0ea5e9;
}

.dialog-actions {
  padding: 16px 24px;
  background: #f8fafc;
}

.action-btn {
  min-width: 120px;
  height: 40px;
  font-weight: 500;
  letter-spacing: 0.02em;
  border-radius: 8px;
  text-transform: none;
}

.primary-btn {
  box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.1), 0 2px 4px -1px rgba(0, 0, 0, 0.06);
}

.primary-btn:hover {
  box-shadow: 0 10px 15px -3px rgba(0, 0, 0, 0.1), 0 4px 6px -2px rgba(0, 0, 0, 0.05);
  transform: translateY(-1px);
}

/* Scrollbar styling */
.invoice-table :deep(.v-table__wrapper)::-webkit-scrollbar {
  width: 6px;
  height: 6px;
}

.invoice-table :deep(.v-table__wrapper)::-webkit-scrollbar-track {
  background: #f1f5f9;
  border-radius: 3px;
}

.invoice-table :deep(.v-table__wrapper)::-webkit-scrollbar-thumb {
  background: #cbd5e1;
  border-radius: 3px;
}

.invoice-table :deep(.v-table__wrapper)::-webkit-scrollbar-thumb:hover {
  background: #94a3b8;
}

/* Checkbox styling */
.invoice-table :deep(.v-checkbox .v-selection-control__wrapper) {
  color: #1976d2 !important;
}

.invoice-table :deep(.v-data-table__td) {
  padding: 12px 16px !important;
}

/* Smooth transitions */
.dialog-card,
.action-btn {
  transition: all 0.2s cubic-bezier(0.4, 0, 0.2, 1);
}

/* Responsive design */
@media (max-width: 768px) {
  .dialog-header {
    padding: 16px;
  }

  .header-content {
    flex-direction: column;
  }

  .header-content .v-icon {
    margin-bottom: 8px;
  }

  .customer-cell {
    flex-direction: column;
    align-items: flex-start;
  }

  .customer-cell .v-avatar {
    margin-bottom: 4px;
  }

  .action-btn {
    min-width: 100px;
    font-size: 0.875rem;
  }
}
</style>