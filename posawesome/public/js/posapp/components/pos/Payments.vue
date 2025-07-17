<template>
  <v-dialog v-model="paymentDialog" max-width="600px" persistent>
    <v-card class="elevation-0" dir="rtl">
      <v-toolbar dense flat color="primary" dark>
        <v-toolbar-title class="text-body-1">
          <v-icon small class="mr-2">mdi-cash-multiple</v-icon>
          {{ __('Payment') }}
        </v-toolbar-title>
        <v-spacer></v-spacer>
        <v-btn icon small @click="back_to_invoice">
          <v-icon>mdi-close</v-icon>
        </v-btn>
      </v-toolbar>

      <v-progress-linear 
        :active="loading" 
        :indeterminate="loading" 
        absolute 
        location="top"
        color="info">
      </v-progress-linear>

      <v-card-text class="pa-2" style="max-height: 70vh; overflow-y: auto;">
        <v-container fluid class="pa-0">
          <v-row dense v-if="invoice_doc">
            <v-col cols="7" class="pr-1">
              <v-card flat outlined class="pa-2">
                <v-row dense class="mb-2">
                  <v-col cols="6">
                    <v-text-field 
                      dense
                      outlined 
                      :label="frappe._('Paid Amount')" 
                      hide-details 
                      :model-value="formatCurrency(total_payments)" 
                      readonly
                      :prefix="currencySymbol(invoice_doc.currency)">
                    </v-text-field>
                  </v-col>
                  <v-col cols="6">
                    <v-text-field 
                      dense
                      outlined 
                      :label="frappe._(diff_lable)" 
                      hide-details
                      :model-value="formatCurrency(diff_payment)" 
                      readonly 
                      :prefix="currencySymbol(invoice_doc.currency)">
                    </v-text-field>
                  </v-col>

                  <v-col cols="6" v-if="diff_payment < 0 && !invoice_doc.is_return">
                    <v-text-field 
                      dense
                      outlined 
                      :label="frappe._('Paid Change')" 
                      v-model="paid_change" 
                      @update:model-value="set_paid_change()"
                      :prefix="currencySymbol(invoice_doc.currency)" 
                      :rules="paid_change_rules" 
                      readonly
                      type="number">
                    </v-text-field>
                  </v-col>

                  <v-col cols="6" v-if="diff_payment < 0 && !invoice_doc.is_return">
                    <v-text-field 
                      dense
                      outlined 
                      :label="frappe._('Credit Change')" 
                      hide-details 
                      :model-value="formatCurrency(credit_change)" 
                      readonly
                      :prefix="currencySymbol(invoice_doc.currency)">
                    </v-text-field>
                  </v-col>
                </v-row>

                <v-divider class="mb-2"></v-divider>

                <div class="text-caption font-weight-medium mb-1">
                  {{ __('Payment Methods') }}
                </div>

                <div>
                  <v-row 
                    dense
                    class="mb-1" 
                    v-for="payment in invoice_doc.payments" 
                    :key="payment.name">
                    <v-col :cols="is_mpesa_c2b_payment(payment) ? 12 : (!is_mpesa_c2b_payment(payment) && (payment.type != 'Phone' || payment.amount == 0 || !request_payment_field)) ? 7 : 5">
                      <v-text-field 
                        v-if="!is_mpesa_c2b_payment(payment)"
                        dense
                        outlined 
                        hide-details
                        v-model.number="payment.amount"
                        @update:model-value="updatePaymentAmount(payment)"
                        @blur="validatePaymentAmounts"
                        :rules="[isNumber]" 
                        :prefix="currencySymbol(invoice_doc.currency)"
                        @focus="set_rest_amount(payment.idx)" 
                        :readonly="invoice_doc.is_return && is_cashback ? true : false"
                        type="number"
                        step="0.01">
                      </v-text-field>
                      <v-btn 
                        v-if="is_mpesa_c2b_payment(payment)"
                        block 
                        small
                        color="success" 
                        @click="mpesa_c2b_dialg(payment)">
                        {{ __(`Get Payments ${payment.mode_of_payment}`) }}
                      </v-btn>
                    </v-col>
                    <v-col 
                      v-if="!is_mpesa_c2b_payment(payment) && (payment.type != 'Phone' || payment.amount == 0 || !request_payment_field)" 
                      :cols="5">
                      <v-btn 
                        block 
                        small
                        color="primary" 
                        @click="set_full_amount(payment.idx)">
                        {{ payment.mode_of_payment }}
                      </v-btn>
                    </v-col>
                    <v-col 
                      v-if="payment.type == 'Phone' && payment.amount > 0 && request_payment_field && !is_mpesa_c2b_payment(payment)" 
                      :cols="2">
                      <v-btn 
                        block 
                        small
                        color="success" 
                        :disabled="payment.amount == 0" 
                        @click="(phone_dialog = true), (payment.amount = flt(payment.amount, 0))">
                        {{ __("Request") }}
                      </v-btn>
                    </v-col>
                  </v-row>
                </div>

                <v-row 
                  dense
                  v-if="invoice_doc && available_pioints_amount > 0 && !invoice_doc.is_return"
                  class="mb-1">
                  <v-col cols="7">
                    <v-text-field 
                      dense
                      outlined 
                      :label="frappe._('Redeem Loyalty Points')" 
                      hide-details 
                      v-model="loyalty_amount"
                      type="number" 
                      :prefix="currencySymbol(invoice_doc.currency)">
                    </v-text-field>
                  </v-col>
                  <v-col cols="5">
                    <v-text-field 
                      dense
                      outlined 
                      :label="frappe._('You can redeem upto')"
                      hide-details 
                      :model-value="formatFloat(available_pioints_amount)"
                      :prefix="currencySymbol(invoice_doc.currency)" 
                      disabled>
                    </v-text-field>
                  </v-col>
                </v-row>

                <v-row 
                  dense
                  v-if="invoice_doc && available_customer_credit > 0 && !invoice_doc.is_return && redeem_customer_credit"
                  class="mb-1">
                  <v-col cols="7">
                    <v-text-field 
                      dense
                      outlined 
                      disabled 
                      :label="frappe._('Redeemed Customer Credit')" 
                      hide-details
                      v-model="redeemed_customer_credit" 
                      type="number"
                      :prefix="currencySymbol(invoice_doc.currency)">
                    </v-text-field>
                  </v-col>
                  <v-col cols="5">
                    <v-text-field 
                      dense
                      outlined 
                      :label="frappe._('You can redeem credit upto')" 
                      hide-details
                      :model-value="formatCurrency(available_customer_credit)" 
                      :prefix="currencySymbol(invoice_doc.currency)"
                      disabled>
                    </v-text-field>
                  </v-col>
                </v-row>

                <div v-if="invoice_doc && available_customer_credit > 0 && !invoice_doc.is_return && redeem_customer_credit">
                  <v-divider class="my-2"></v-divider>
                  <div class="text-caption font-weight-medium mb-1">
                    {{ __('Customer Credit Details') }}
                  </div>
                  <v-row dense v-for="(row, idx) in customer_credit_dict" :key="idx" class="mb-1">
                    <v-col cols="4">
                      <div class="pa-1 text-caption">{{ row.credit_origin }}</div>
                    </v-col>
                    <v-col cols="4">
                      <v-text-field 
                        dense
                        outlined 
                        :label="frappe._('Available Credit')"
                        hide-details 
                        :model-value="formatCurrency(row.total_credit)" 
                        disabled
                        :prefix="currencySymbol(invoice_doc.currency)">
                      </v-text-field>
                    </v-col>
                    <v-col cols="4">
                      <v-text-field 
                        dense
                        outlined 
                        :label="frappe._('Redeem Credit')"
                        hide-details 
                        type="number" 
                        v-model="row.credit_to_redeem"
                        :prefix="currencySymbol(invoice_doc.currency)">
                      </v-text-field>
                    </v-col>
                  </v-row>
                </div>

                <v-divider class="my-2"></v-divider>

                <v-row dense>
                  <v-col cols="6" v-if="pos_profile.posa_allow_write_off_change && diff_payment > 0 && !invoice_doc.is_return">
                    <v-switch 
                      dense
                      v-model="is_write_off_change" 
                      :label="frappe._('Write Off Difference Amount')"
                      hide-details
                      class="mt-0">
                    </v-switch>
                  </v-col>
                  <v-col cols="6" v-if="invoice_doc.is_return && pos_profile.use_cashback">
                    <v-switch 
                      dense
                      v-model="is_cashback" 
                      :label="frappe._('Cashback?')"
                      hide-details
                      class="mt-0">
                    </v-switch>
                  </v-col>
                  <v-col cols="6" v-if="!invoice_doc.is_return && pos_profile.use_customer_credit">
                    <v-switch 
                      dense
                      v-model="redeem_customer_credit" 
                      :label="frappe._('Use Customer Credit')"
                      hide-details
                      class="mt-0"
                      @update:model-value="get_available_credit($event)">
                    </v-switch>
                  </v-col>
                </v-row>
              </v-card>
            </v-col>

            <v-col cols="5" class="pl-1">
              <v-card flat color="primary" dark class="pa-2">
                <div class="text-caption font-weight-bold mb-2 text-white">
                  {{ __('Invoice Summary') }}
                </div>

                <v-card flat class="pa-2 mb-2">
                  <v-row no-gutters>
                    <v-col cols="12" class="d-flex justify-space-between align-center py-1">
                      <span class="text-caption">{{ __('Net Total') }}</span>
                      <span class="text-caption font-weight-medium">
                        {{ formatCurrency(invoice_doc.net_total) }} {{ currencySymbol(invoice_doc.currency) }}
                      </span>
                    </v-col>
                    <v-divider class="my-1"></v-divider>
                    <v-col cols="12" class="d-flex justify-space-between align-center py-1">
                      <span class="text-caption">{{ __('Tax and Charges') }}</span>
                      <span class="text-caption font-weight-medium">
                        {{ formatCurrency(invoice_doc.total_taxes_and_charges) }} {{ currencySymbol(invoice_doc.currency) }}
                      </span>
                    </v-col>
                    <v-divider class="my-1" v-if="invoice_doc.discount_amount"></v-divider>
                    <v-col cols="12" class="d-flex justify-space-between align-center py-1" v-if="invoice_doc.discount_amount">
                      <span class="text-caption">{{ __('Discount Amount') }}</span>
                      <span class="text-caption font-weight-medium text-success">
                        -{{ formatCurrency(invoice_doc.discount_amount) }} {{ currencySymbol(invoice_doc.currency) }}
                      </span>
                    </v-col>
                  </v-row>
                </v-card>

                <v-card flat class="pa-2 bg-primary-darken-1">
                  <v-row no-gutters>
                    <v-col cols="12" class="d-flex justify-space-between align-center">
                      <span class="text-body-2 font-weight-bold text-white">{{ __('Grand Total') }}</span>
                      <span class="text-h6 font-weight-bold text-white">
                        {{ formatCurrency(invoice_doc.grand_total) }} {{ currencySymbol(invoice_doc.currency) }}
                      </span>
                    </v-col>
                    <v-col cols="12" class="d-flex justify-space-between align-center mt-1" v-if="invoice_doc.rounded_total">
                      <span class="text-caption text-white-70">{{ __('Rounded Total') }}</span>
                      <span class="text-body-2 font-weight-medium text-white">
                        {{ formatCurrency(invoice_doc.rounded_total) }} {{ currencySymbol(invoice_doc.currency) }}
                      </span>
                    </v-col>
                  </v-row>
                </v-card>

                <v-divider class="my-2"></v-divider>

                <v-row dense v-if="pos_profile && pos_profile.posa_allow_credit_sale && !invoice_doc.is_return">
                  <v-col cols="12">
                    <v-switch 
                      dense
                      v-model="is_credit_sale" 
                      :label="frappe._('Credit Sale?')"
                      hide-details
                      class="mt-0">
                    </v-switch>
                  </v-col>
                  <v-col cols="12" v-if="is_credit_sale" class="mt-2">
                    <v-menu 
                      ref="date_menu" 
                      v-model="date_menu" 
                      :close-on-content-click="false" 
                      transition="scale-transition">
                      <template v-slot:activator="{ props }">
                        <v-text-field 
                          dense
                          v-model="invoice_doc.due_date" 
                          :label="frappe._('Due Date')" 
                          readonly 
                          outlined
                          hide-details 
                          v-bind="props" 
                          prepend-inner-icon="mdi-calendar">
                        </v-text-field>
                      </template>
                      <v-date-picker 
                        v-model="credit_sales_due_date" 
                        no-title 
                        scrollable 
                        color="primary"
                        :min="frappe.datetime.now_date()" 
                        @input="date_menu = false">
                      </v-date-picker>
                    </v-menu>
                  </v-col>
                </v-row>

                <v-divider class="my-2"></v-divider>

                <v-row dense>
                  <v-col cols="12" v-if="pos_profile.posa_allow_sales_order && invoiceType == 'Order'">
                    <v-menu 
                      ref="order_delivery_date" 
                      v-model="order_delivery_date" 
                      :close-on-content-click="false"
                      transition="scale-transition">
                      <template v-slot:activator="{ props }">
                        <v-text-field 
                          dense
                          v-model="invoice_doc.posa_delivery_date" 
                          :label="frappe._('Delivery Date')" 
                          readonly
                          outlined 
                          bg-color="white" 
                          clearable 
                          hide-details
                          v-bind="props"
                          prepend-inner-icon="mdi-calendar">
                        </v-text-field>
                      </template>
                      <v-date-picker 
                        :v-model="new Date(invoice_doc.posa_delivery_date)" 
                        no-title 
                        scrollable 
                        color="primary"
                        :min="frappe.datetime.now_date()" 
                        @input="order_delivery_date = false">
                      </v-date-picker>
                    </v-menu>
                  </v-col>

                  <v-col cols="12" v-if="invoice_doc.posa_delivery_date" class="mt-2">
                    <v-autocomplete 
                      dense
                      clearable 
                      auto-select-first 
                      outlined 
                      :label="frappe._('Address')" 
                      v-model="invoice_doc.shipping_address_name" 
                      :items="addresses"
                      item-title="address_title" 
                      item-value="name" 
                      bg-color="white" 
                      no-data-text="Address not found"
                      hide-details 
                      :customFilter="addressFilter" 
                      append-icon="mdi-plus" 
                      @click:append="new_address">
                      <template v-slot:item="{ props, item }">
                        <v-list-item v-bind="props">
                          <v-list-item-title class="text-primary text-caption">
                            <div v-html="item.raw.address_title"></div>
                          </v-list-item-title>
                          <v-list-item-title class="text-caption">
                            <div v-html="item.raw.address_line1"></div>
                          </v-list-item-title>
                          <v-list-item-subtitle v-if="item.raw.custoaddress_line2mer_name" class="text-caption">
                            <div v-html="item.raw.address_line2"></div>
                          </v-list-item-subtitle>
                          <v-list-item-subtitle v-if="item.raw.city" class="text-caption">
                            <div v-html="item.raw.city"></div>
                          </v-list-item-subtitle>
                          <v-list-item-subtitle v-if="item.raw.state" class="text-caption">
                            <div v-html="item.raw.state"></div>
                          </v-list-item-subtitle>
                          <v-list-item-subtitle v-if="item.raw.country" class="text-caption">
                            <div v-html="item.raw.mobile_no"></div>
                          </v-list-item-subtitle>
                          <v-list-item-subtitle v-if="item.raw.address_type" class="text-caption">
                            <div v-html="item.raw.address_type"></div>
                          </v-list-item-subtitle>
                        </v-list-item>
                      </template>
                    </v-autocomplete>
                  </v-col>

                  <v-col cols="12" v-if="pos_profile.posa_display_additional_notes" class="mt-2">
                    <v-textarea 
                      outlined 
                      dense
                      bg-color="white" 
                      clearable 
                      auto-grow 
                      rows="2" 
                      :label="frappe._('Additional Notes')" 
                      v-model="invoice_doc.posa_notes"
                      :model-value="invoice_doc.posa_notes">
                    </v-textarea>
                  </v-col>
                </v-row>

                <div v-if="pos_profile.posa_allow_customer_purchase_order">
                  <v-divider class="my-2"></v-divider>
                  <v-row dense>
                    <v-col cols="6">
                      <v-text-field 
                        dense
                        v-model="invoice_doc.po_no" 
                        :label="frappe._('Purchase Order')" 
                        outlined
                        bg-color="white" 
                        clearable 
                        hide-details>
                      </v-text-field>
                    </v-col>
                    <v-col cols="6">
                      <v-menu 
                        ref="po_date_menu" 
                        v-model="po_date_menu" 
                        :close-on-content-click="false"
                        transition="scale-transition">
                        <template v-slot:activator="{ props }">
                          <v-text-field 
                            dense
                            v-model="invoice_doc.po_date" 
                            :label="frappe._('PO Date')" 
                            readonly
                            outlined 
                            hide-details 
                            v-bind="props">
                          </v-text-field>
                        </template>
                        <v-date-picker 
                          v-model="invoice_doc.po_date" 
                          no-title 
                          scrollable 
                          color="primary"
                          @input="po_date_menu = false">
                        </v-date-picker>
                      </v-menu>
                    </v-col>
                  </v-row>
                </div>
              </v-card>
            </v-col>
          </v-row>
        </v-container>
      </v-card-text>

      <v-divider></v-divider>

      <v-card-actions class="pa-3">
        <v-btn 
          large
          text
          color="error" 
          @click="back_to_invoice">
          {{ __("Cancel Payment") }}
          <v-chip x-small outlined class="ml-1">F9</v-chip>
        </v-btn>
        <v-spacer></v-spacer>
        <v-btn 
          large
          color="primary" 
          @click="submit" 
          :disabled="vaildatPayment">
          {{ __("Submit") }}
          <v-chip x-small outlined class="ml-1">F10</v-chip>
        </v-btn>
        <v-btn 
          large
          color="success" 
          @click="submit(undefined, false, true)"
          :disabled="vaildatPayment">
          {{ __("Submit & Print") }}
          <v-chip x-small outlined class="ml-1">F12</v-chip>
        </v-btn>
      </v-card-actions>
    </v-card>

    <v-dialog v-model="phone_dialog" max-width="400px">
      <v-card>
        <v-card-title>
          <span class="text-h5 text-primary">{{ __("Confirm Mobile Number") }}</span>
        </v-card-title>
        <v-card-text class="pa-0">
          <v-container>
            <v-text-field 
              dense
              outlined 
              :label="frappe._('Mobile Number')"
              hide-details 
              v-model="invoice_doc.contact_mobile" 
              type="number">
            </v-text-field>
          </v-container>
        </v-card-text>
        <v-card-actions>
          <v-spacer></v-spacer>
          <v-btn 
            color="error" 
            theme="dark" 
            @click="phone_dialog = false">
            {{ __("Close") }}
          </v-btn>
          <v-btn 
            color="primary" 
            theme="dark" 
            @click="request_payment">
            {{ __("Request") }}
          </v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>
  </v-dialog>
</template>

<script>
import format from "../../format";
export default {
  mixins: [format],
  data: () => ({
    paymentDialog: false,
    loading: false,
    pos_profile: "",
    invoice_doc: "",
    loyalty_amount: 0,
    credit_sales_due_date: new Date(frappe.datetime.now_date()),
    is_credit_sale: 0,
    is_write_off_change: 0,
    date_menu: false,
    po_date_menu: false,
    addresses: [],
    sales_persons: [],
    sales_person: "",
    paid_change: 0,
    order_delivery_date: false,
    paid_change_rules: [],
    is_return: false,
    is_cashback: true,
    redeem_customer_credit: false,
    customer_credit_dict: [],
    phone_dialog: false,
    invoiceType: "Invoice",
    pos_settings: "",
    customer_info: "",
    mpesa_modes: [],
  }),

  methods: {
    back_to_invoice() {
      this.paymentDialog = false;
      this.eventBus.emit("show_payment", "false");
      this.eventBus.emit("set_customer_readonly", false);
    },
    submit(event, payment_received = false, print = false) {
      if (!this.invoice_doc.is_return && this.total_payments < 0) {
        this.eventBus.emit("show_message", {
          title: `Payments not correct`,
          color: "error",
        });
        frappe.utils.play_sound("error");
        return;
      }
      
      const totalAmount = this.invoice_doc.rounded_total || this.invoice_doc.grand_total;
      const tolerance = 0.01;
      
      if (!this.is_credit_sale && Math.abs(this.total_payments - totalAmount) > tolerance && this.total_payments < totalAmount) {
        this.eventBus.emit("show_message", {
          title: this.__(`Payment amount (${this.formatCurrency(this.total_payments)}) does not match invoice total (${this.formatCurrency(totalAmount)})`),
          color: "error",
        });
        frappe.utils.play_sound("error");
        return;
      }
      
      let phone_payment_is_valid = true;
      if (!payment_received) {
        this.invoice_doc.payments.forEach((payment) => {
          if (
            payment.type == "Phone" &&
            ![0, "0", "", null, undefined].includes(payment.amount)
          ) {
            phone_payment_is_valid = false;
          }
        });
        if (!phone_payment_is_valid) {
          this.eventBus.emit("show_message", {
            title: __(
              "Please request phone payment or use other payment method"
            ),
            color: "error",
          });
          frappe.utils.play_sound("error");
          console.error("phone payment not requested");
          return;
        }
      }

      if (
        !this.is_credit_sale && !this.pos_profile.posa_allow_partial_payment &&
        this.total_payments <
        (this.invoice_doc.rounded_total || this.invoice_doc.grand_total)
      ) {
        this.eventBus.emit("show_message", {
          title: `The amount paid is not complete`,
          color: "error",
        });
        frappe.utils.play_sound("error");
        return;
      }

      if (
        this.pos_profile.posa_allow_partial_payment &&
        !this.pos_profile.posa_allow_credit_sale &&
        this.total_payments == 0
      ) {
        this.eventBus.emit("show_message", {
          title: `Please enter the amount paid`,
          color: "error",
        });
        frappe.utils.play_sound("error");
        return;
      }

      if (!this.paid_change) this.paid_change = 0;

      if (this.paid_change > -this.diff_payment) {
        this.eventBus.emit("show_message", {
          title: `Paid change can not be greater than total change!`,
          color: "error",
        });
        frappe.utils.play_sound("error");
        return;
      }

      let total_change = this.flt(
        this.flt(this.paid_change) + this.flt(-this.credit_change)
      );

      if (this.is_cashback && total_change != -this.diff_payment) {
        this.eventBus.emit("show_message", {
          title: `Error in change calculations!`,
          color: "error",
        });
        frappe.utils.play_sound("error");
        return;
      }

      let credit_calc_check = this.customer_credit_dict.filter((row) => {
        if (flt(row.credit_to_redeem))
          return flt(row.credit_to_redeem) > flt(row.total_credit);
        else return false;
      });

      if (credit_calc_check.length > 0) {
        this.eventBus.emit("show_message", {
          title: `redeamed credit can not greater than its total.`,
          color: "error",
        });
        frappe.utils.play_sound("error");
        return;
      }

      if (
        !this.invoice_doc.is_return &&
        this.redeemed_customer_credit >
        (this.invoice_doc.rounded_total || this.invoice_doc.grand_total)
      ) {
        this.eventBus.emit("show_message", {
          title: `can not redeam customer credit more than invoice total`,
          color: "error",
        });
        frappe.utils.play_sound("error");
        return;
      }
      this.is_sucessful_invoice = this.submit_invoice(print);
    },
    submit_invoice(print) {
      let totalPayedAmount = 0;
      this.invoice_doc.payments.forEach((payment) => {
        payment.amount = flt(payment.amount);
        totalPayedAmount += payment.amount;
      });
      if (this.invoice_doc.is_return && totalPayedAmount == 0) {
        this.invoice_doc.is_pos = 0;
      }
      if (this.customer_credit_dict.length) {
        this.customer_credit_dict.forEach((row) => {
          row.credit_to_redeem = flt(row.credit_to_redeem);
        });
      }
      let data = {};
      data["total_change"] = !this.invoice_doc.is_return
        ? -this.diff_payment
        : 0;
      data["paid_change"] = !this.invoice_doc.is_return ? this.paid_change : 0;
      data["credit_change"] = -this.credit_change;
      data["redeemed_customer_credit"] = this.redeemed_customer_credit;
      data["customer_credit_dict"] = this.customer_credit_dict;
      data["is_cashback"] = this.is_cashback;

      const vm = this;
      frappe.call({
        method: "posawesome.posawesome.api.posapp.submit_invoice",
        args: {
          data: data,
          invoice: this.invoice_doc,
        },
        async: false,
        callback: function (r) {
          if (!r?.message) {
            vm.eventBus.emit("show_message", {
              title: `Error submitting invoice`,
              color: "error",
            });
            return;
          }
          if (print) {
            vm.load_print_page();
          }
          vm.customer_credit_dict = [];
          vm.redeem_customer_credit = false;
          vm.is_cashback = true;
          vm.sales_person = "";

          vm.eventBus.emit("set_last_invoice", vm.invoice_doc.name);
          vm.eventBus.emit("show_message", {
            title: `Invoice ${r.message.name} is Submited`,
            color: "success",
          });
          frappe.utils.play_sound("submit");
          vm.addresses = [];
          vm.eventBus.emit("clear_invoice");
          vm.back_to_invoice();
          return;
        }
      });
      console.log(this.is_sucessful_invoice)
    },
    set_full_amount(idx) {
      this.invoice_doc.payments.forEach((payment) => {
        payment.amount =
          payment.idx == idx
            ? this.flt(this.invoice_doc.rounded_total || this.invoice_doc.grand_total, this.currency_precision)
            : 0;
      });
    },
    set_specific_amount(idx) {
      const targetPayment = this.invoice_doc.payments.find(p => p.idx === idx);
      if (targetPayment && targetPayment.amount === 0) {
        if (this.diff_payment > 0) {
          targetPayment.amount = this.flt(this.diff_payment, this.currency_precision);
        } else {
          this.invoice_doc.payments.forEach((payment) => {
            payment.amount = payment.idx === idx 
              ? this.flt(this.invoice_doc.rounded_total || this.invoice_doc.grand_total, this.currency_precision)
              : 0;
          });
        }
      } else {
        this.invoice_doc.payments.forEach((payment) => {
          payment.amount = payment.idx === idx 
            ? this.flt(this.invoice_doc.rounded_total || this.invoice_doc.grand_total, this.currency_precision)
            : 0;
        });
      }
    },
    set_rest_amount(idx) {
      const targetPayment = this.invoice_doc.payments.find(p => p.idx === idx);
      if (targetPayment && targetPayment.amount == 0 && this.diff_payment > 0) {
        const totalAmount = this.invoice_doc.rounded_total || this.invoice_doc.grand_total;
        let otherPaymentsTotal = 0;
        
        this.invoice_doc.payments.forEach((p) => {
          if (p.idx !== idx) {
            otherPaymentsTotal += this.flt(p.amount || 0);
          }
        });
        
        otherPaymentsTotal += this.flt(this.loyalty_amount || 0);
        otherPaymentsTotal += this.flt(this.redeemed_customer_credit || 0);
        
        const remaining = this.flt(totalAmount - otherPaymentsTotal, this.currency_precision);
        if (remaining > 0) {
          targetPayment.amount = remaining;
        }
      }
    },
    updatePaymentAmount(payment) {
      payment.amount = parseFloat(payment.amount || 0);
      if (isNaN(payment.amount)) {
        payment.amount = 0;
      }
      payment.amount = this.flt(payment.amount, this.currency_precision);
      
      const totalAmount = this.invoice_doc.rounded_total || this.invoice_doc.grand_total;
      let currentTotal = 0;
      let lastEmptyPayment = null;
      let emptyPaymentsCount = 0;
      
      this.invoice_doc.payments.forEach((p) => {
        if (p.idx !== payment.idx) {
          if (p.amount === 0 || p.amount === '') {
            emptyPaymentsCount++;
            lastEmptyPayment = p;
          } else {
            currentTotal += this.flt(p.amount || 0, this.currency_precision);
          }
        } else {
          currentTotal += this.flt(payment.amount || 0, this.currency_precision);
        }
      });
      
      currentTotal += this.flt(this.loyalty_amount || 0, this.currency_precision);
      currentTotal += this.flt(this.redeemed_customer_credit || 0, this.currency_precision);
      
      if (currentTotal > totalAmount) {
        this.eventBus.emit("show_message", {
          title: this.__('Payment amount exceeds invoice total!'),
          color: "error",
        });
        payment.amount = this.flt(totalAmount - (currentTotal - payment.amount), this.currency_precision);
        if (payment.amount < 0) payment.amount = 0;
      } else if (emptyPaymentsCount === 1 && lastEmptyPayment && !this.is_credit_sale) {
        const remaining = this.flt(totalAmount - currentTotal, this.currency_precision);
        if (remaining > 0) {
          lastEmptyPayment.amount = remaining;
        }
      }
    },
    validatePaymentAmounts() {
      const totalAmount = this.invoice_doc.rounded_total || this.invoice_doc.grand_total;
      const totalPaid = this.total_payments;
      
      if (Math.abs(totalPaid - totalAmount) > 0.01 && !this.is_credit_sale && totalPaid > 0) {
        if (totalPaid < totalAmount) {
          this.eventBus.emit("show_message", {
            title: this.__(`Payment amount is less than invoice total by ${this.formatCurrency(totalAmount - totalPaid)}`),
            color: "warning",
          });
        }
      }
    },
    clear_all_amounts() {
      this.invoice_doc.payments.forEach((payment) => {
        payment.amount = 0;
      });
    },
    load_print_page() {
      const print_format =
        this.pos_profile.print_format_for_online ||
        this.pos_profile.print_format;
      const letter_head = this.pos_profile.letter_head || 0;
      const url =
        frappe.urllib.get_base_url() +
        "/printview?doctype=Sales%20Invoice&name=" +
        this.invoice_doc.name +
        "&trigger_print=1" +
        "&format=" +
        print_format +
        "&no_letterhead=" +
        letter_head;
      
      let printFrame = document.getElementById('print-frame');
      if (printFrame) {
        printFrame.remove();
      }
      
      printFrame = document.createElement('iframe');
      printFrame.id = 'print-frame';
      printFrame.style.position = 'absolute';
      printFrame.style.width = '0';
      printFrame.style.height = '0';
      printFrame.style.border = '0';
      printFrame.style.left = '-9999px';
      printFrame.style.top = '-9999px';
      printFrame.style.visibility = 'hidden';
      document.body.appendChild(printFrame);
      
      printFrame.onload = function() {
        printFrame.contentWindow.focus();
        printFrame.contentWindow.print();
        
        setTimeout(() => {
          if (printFrame && printFrame.parentNode) {
            printFrame.remove();
          }
        }, 100);
      };
      
      printFrame.src = url;
    },
    validate_due_date() {
      const today = frappe.datetime.now_date();
      const parse_today = Date.parse(today);
      const new_date = Date.parse(this.invoice_doc.due_date);
      if (new_date < parse_today) {
        setTimeout(() => {
          this.invoice_doc.due_date = today;
        }, 0);
      }
    },
    shortPay(e) {
      if (e.key === "x" && (e.ctrlKey || e.metaKey)) {
        e.preventDefault();
        this.submit();
      }
      
      if (e.key === "F12") {
        e.preventDefault();
        if (!this.vaildatPayment) {
          this.submit(undefined, false, true);
        }
      }
      
      if (e.key === "F10") {
        e.preventDefault();
        if (!this.vaildatPayment) {
          this.submit();
        }
      }
      
      if (e.key === "F9") {
        e.preventDefault();
        this.back_to_invoice();
      }
      
      if (e.key === "Escape") {
        e.preventDefault();
        this.back_to_invoice();
      }
    },
    set_paid_change() {
      if (!this.paid_change) this.paid_change = 0;

      this.paid_change_rules = [];
      let change = -this.diff_payment;
      if (this.paid_change > change) {
        this.paid_change_rules = [
          "Paid change can not be greater than total change!",
        ];
        this.credit_change = 0;
      }
    },
    get_available_credit(e) {
      this.clear_all_amounts();
      if (e) {
        frappe
          .call("posawesome.posawesome.api.posapp.get_available_credit", {
            customer: this.invoice_doc.customer,
            company: this.pos_profile.company,
          })
          .then((r) => {
            const data = r.message;
            if (data.length) {
              const amount =
                this.invoice_doc.rounded_total || this.invoice_doc.grand_total;
              let remainAmount = amount;

              data.forEach((row) => {
                if (remainAmount > 0) {
                  if (remainAmount >= row.total_credit) {
                    row.credit_to_redeem = row.total_credit;
                    remainAmount = remainAmount - row.total_credit;
                  } else {
                    row.credit_to_redeem = remainAmount;
                    remainAmount = 0;
                  }
                } else {
                  row.credit_to_redeem = 0;
                }
              });

              this.customer_credit_dict = data;
            } else {
              this.customer_credit_dict = [];
            }
          });
      } else {
        this.customer_credit_dict = [];
      }
    },
    get_addresses() {
      const vm = this;
      if (!vm.invoice_doc) {
        return;
      }
      frappe.call({
        method: "posawesome.posawesome.api.posapp.get_customer_addresses",
        args: { customer: vm.invoice_doc.customer },
        async: true,
        callback: function (r) {
          if (!r.exc) {
            vm.addresses = r.message;
          } else {
            vm.addresses = [];
          }
        },
      });
    },
    addressFilter(item, queryText, itemText) {
      const textOne = item.address_title
        ? item.address_title.toLowerCase()
        : "";
      const textTwo = item.address_line1
        ? item.address_line1.toLowerCase()
        : "";
      const textThree = item.address_line2
        ? item.address_line2.toLowerCase()
        : "";
      const textFour = item.city ? item.city.toLowerCase() : "";
      const textFifth = item.name.toLowerCase();
      const searchText = queryText.toLowerCase();
      return (
        textOne.indexOf(searchText) > -1 ||
        textTwo.indexOf(searchText) > -1 ||
        textThree.indexOf(searchText) > -1 ||
        textFour.indexOf(searchText) > -1 ||
        textFifth.indexOf(searchText) > -1
      );
    },
    new_address() {
      this.eventBus.emit("open_new_address", this.invoice_doc.customer);
    },
    get_sales_person_names() {
      const vm = this;
      if (
        vm.pos_profile.posa_local_storage &&
        localStorage.sales_persons_storage
      ) {
        vm.sales_persons = JSON.parse(
          localStorage.getItem("sales_persons_storage")
        );
      }
      frappe.call({
        method: "posawesome.posawesome.api.posapp.get_sales_person_names",
        callback: function (r) {
          if (r.message) {
            vm.sales_persons = r.message;
            if (vm.pos_profile.posa_local_storage) {
              localStorage.setItem("sales_persons_storage", "");
              localStorage.setItem(
                "sales_persons_storage",
                JSON.stringify(r.message)
              );
            }
          }
        },
      });
    },
    salesPersonFilter(itemText, queryText, itemRow) {
      const item = itemRow.raw;
      const textOne = item.sales_person_name
        ? item.sales_person_name.toLowerCase()
        : "";
      const textTwo = item.name.toLowerCase();
      const searchText = queryText.toLowerCase();

      return (
        textOne.indexOf(searchText) > -1 || textTwo.indexOf(searchText) > -1
      );
    },
    request_payment() {
      this.phone_dialog = false;
      const vm = this;
      if (!this.invoice_doc.contact_mobile) {
        this.eventBus.emit("show_message", {
          title: __(`Pleas Set Customer Mobile Number`),
          color: "error",
        });
        this.eventBus.emit("open_edit_customer");
        this.back_to_invoice();
        return;
      }
      this.eventBus.emit("freeze", {
        title: __(`Waiting for payment... `),
      });
      this.invoice_doc.payments.forEach((payment) => {
        payment.amount = flt(payment.amount);
      });
      let formData = { ...this.invoice_doc };
      formData["total_change"] = -this.diff_payment;
      formData["paid_change"] = this.paid_change;
      formData["credit_change"] = -this.credit_change;
      formData["redeemed_customer_credit"] = this.redeemed_customer_credit;
      formData["customer_credit_dict"] = this.customer_credit_dict;
      formData["is_cashback"] = this.is_cashback;

      frappe
        .call({
          method: "posawesome.posawesome.api.posapp.update_invoice",
          args: {
            data: formData,
          },
          async: false,
          callback: function (r) {
            if (r.message) {
              vm.invoice_doc = r.message;
            }
          },
        })
        .then(() => {
          frappe
            .call({
              method: "posawesome.posawesome.api.posapp.create_payment_request",
              args: {
                doc: vm.invoice_doc,
              },
            })
            .fail(() => {
              this.eventBus.emit("unfreeze");
              this.eventBus.emit("show_message", {
                title: __(`Payment request failed`),
                color: "error",
              });
            })
            .then(({ message }) => {
              const payment_request_name = message.name;
              setTimeout(() => {
                frappe.db
                  .get_value("Payment Request", payment_request_name, [
                    "status",
                    "grand_total",
                  ])
                  .then(({ message }) => {
                    if (message.status != "Paid") {
                      this.eventBus.emit("unfreeze");
                      this.eventBus.emit("show_message", {
                        title: __(
                          `Payment Request took too long to respond. Please try requesting for payment again`
                        ),
                        color: "error",
                      });
                    } else {
                      this.eventBus.emit("unfreeze");
                      this.eventBus.emit("show_message", {
                        title: __("Payment of {0} received successfully.", [
                          vm.formatCurrency(
                            message.grand_total,
                            vm.invoice_doc.currency,
                            0
                          ),
                        ]),
                        color: "success",
                      });
                      frappe.db
                        .get_doc("Sales Invoice", vm.invoice_doc.name)
                        .then((doc) => {
                          vm.invoice_doc = doc;
                          vm.submit(null, true);
                        });
                    }
                  });
              }, 30000);
            });
        });
    },
    get_mpesa_modes() {
      const vm = this;
      frappe.call({
        method: "posawesome.posawesome.api.m_pesa.get_mpesa_mode_of_payment",
        args: { company: vm.pos_profile.company },
        async: true,
        callback: function (r) {
          if (!r.exc) {
            vm.mpesa_modes = r.message;
          } else {
            vm.mpesa_modes = [];
          }
        },
      });
    },
    is_mpesa_c2b_payment(payment) {
      if (
        this.mpesa_modes.includes(payment.mode_of_payment) &&
        payment.type == "Bank"
      ) {
        payment.amount = 0;
        return true;
      } else {
        return false;
      }
    },
    mpesa_c2b_dialg(payment) {
      const data = {
        company: this.pos_profile.company,
        mode_of_payment: payment.mode_of_payment,
        customer: this.invoice_doc.customer,
      };
      this.eventBus.emit("open_mpesa_payments", data);
    },
    set_mpesa_payment(payment) {
      this.pos_profile.use_customer_credit = 1;
      this.redeem_customer_credit = true;
      const invoiceAmount =
        this.invoice_doc.rounded_total || this.invoice_doc.grand_total;
      let amount =
        payment.unallocated_amount > invoiceAmount
          ? invoiceAmount
          : payment.unallocated_amount;
      if (amount < 0 || !amount) amount = 0;
      const advance = {
        type: "Advance",
        credit_origin: payment.name,
        total_credit: flt(payment.unallocated_amount),
        credit_to_redeem: flt(amount),
      };
      this.clear_all_amounts();
      this.customer_credit_dict.push(advance);
    },
  },

  computed: {
    total_payments() {
      let total = parseFloat(this.invoice_doc.loyalty_amount);
      if (this.invoice_doc && this.invoice_doc.payments) {
        this.invoice_doc.payments.forEach((payment) => {
          total += this.flt(payment.amount);
        });
      }

      total += this.flt(this.redeemed_customer_credit);

      if (!this.is_cashback && this.invoice_doc.is_return) total = 0;

      return this.flt(total, this.currency_precision);
    },
    diff_payment() {
      let diff_payment = this.flt(
        (this.invoice_doc.rounded_total || this.invoice_doc.grand_total) -
        this.total_payments,
        this.currency_precision
      );
      this.paid_change = -diff_payment;
      return diff_payment;
    },
    credit_change() {
      let change = -this.diff_payment;
      if (this.paid_change > change) return 0;
      return this.flt(this.paid_change - change, this.currency_precision);
    },
    diff_lable() {
      let lable = this.diff_payment < 0 ? "Change" : "To Be Paid";
      return lable;
    },
    available_pioints_amount() {
      let amount = 0;
      if (this.customer_info.loyalty_points) {
        amount =
          this.customer_info.loyalty_points *
          this.customer_info.conversion_factor;
      }
      return amount;
    },
    available_customer_credit() {
      let total = 0;
      this.customer_credit_dict.map((row) => {
        total += row.total_credit;
      });

      return total;
    },
    redeemed_customer_credit() {
      let total = 0;
      this.customer_credit_dict.map((row) => {
        if (flt(row.credit_to_redeem)) total += flt(row.credit_to_redeem);
        else row.credit_to_redeem = 0;
      });

      return total;
    },
    vaildatPayment() {
      if (this.pos_profile.posa_allow_sales_order) {
        if (
          this.invoiceType == "Order" &&
          !this.invoice_doc.posa_delivery_date
        ) {
          return true;
        } else {
          return false;
        }
      } else {
        return false;
      }
    },
    request_payment_field() {
      let res = false;
      if (!this.pos_settings || this.pos_settings.invoice_fields.length == 0) {
        res = false;
      } else {
        this.pos_settings.invoice_fields.forEach((el) => {
          if (
            el.fieldtype == "Button" &&
            el.fieldname == "request_for_payment"
          ) {
            res = true;
          }
        });
      }
      return res;
    },
  },

  mounted: function () {
    this.$nextTick(function () {
      this.eventBus.on("send_invoice_doc_payment", (invoice_doc) => {
        this.invoice_doc = invoice_doc;
        this.paymentDialog = true;
        const default_payment = this.invoice_doc.payments.find(
          (payment) => payment.default == 1
        );
        this.is_credit_sale = 0;
        this.is_write_off_change = 0;
        if (default_payment && !invoice_doc.is_return) {
          default_payment.amount = this.flt(
            invoice_doc.rounded_total || invoice_doc.grand_total,
            this.currency_precision
          );
        }
        if (invoice_doc.is_return) {
          this.is_return = true;
          invoice_doc.payments.forEach((payment) => {
            payment.amount = 0;
            payment.base_amount = 0;
          });
        }
        this.loyalty_amount = 0;
        this.get_addresses();
        this.get_sales_person_names();
      });
      this.eventBus.on("register_pos_profile", (data) => {
        this.pos_profile = data.pos_profile;
        this.get_mpesa_modes();
      });
      this.eventBus.on("add_the_new_address", (data) => {
        this.addresses.push(data);
        this.$forceUpdate();
      });
      this.eventBus.on("update_invoice_type", (data) => {
        this.invoiceType = data;
        if (this.invoice_doc && data != "Order") {
          this.invoice_doc.posa_delivery_date = null;
          this.invoice_doc.posa_notes = null;
          this.invoice_doc.shipping_address_name = null;
        }
      });
    });
    this.eventBus.on("update_customer", (customer) => {
      if (this.customer != customer) {
        this.customer_credit_dict = [];
        this.redeem_customer_credit = false;
        this.is_cashback = true;
      }
    });
    this.eventBus.on("set_pos_settings", (data) => {
      this.pos_settings = data;
    });
    this.eventBus.on("set_customer_info_to_edit", (data) => {
      this.customer_info = data;
    });
    this.eventBus.on("set_mpesa_payment", (data) => {
      this.set_mpesa_payment(data);
    });
  },
  created() {
    document.addEventListener("keydown", this.shortPay.bind(this));
  },
  beforeUnmount() {
    evntBus.$off("send_invoice_doc_payment");
    evntBus.$off("register_pos_profile");
    evntBus.$off("add_the_new_address");
    evntBus.$off("update_invoice_type");
    evntBus.$off("update_customer");
    evntBus.$off("set_pos_settings");
    evntBus.$off("set_customer_info_to_edit");
    evntBus.$off("update_invoice_coupons");
    evntBus.$off("set_mpesa_payment");
  },

  unmounted() {
    document.removeEventListener("keydown", this.shortPay);
  },

  watch: {
    loyalty_amount(value) {
      if (value > this.available_pioints_amount) {
        this.invoice_doc.loyalty_amount = 0;
        this.invoice_doc.redeem_loyalty_points = 0;
        this.invoice_doc.loyalty_points = 0;
        this.eventBus.emit("show_message", {
          title: `Loyalty Amount can not be more then ${this.available_pioints_amount}`,
          color: "error",
        });
      } else {
        this.invoice_doc.loyalty_amount = this.flt(this.loyalty_amount);
        this.invoice_doc.redeem_loyalty_points = 1;
        this.invoice_doc.loyalty_points =
          this.flt(this.loyalty_amount) / this.customer_info.conversion_factor;
      }
    },
    is_credit_sale(value) {
      if (value) {
        this.invoice_doc.payments.forEach((payment) => {
          payment.amount = 0;
          payment.base_amount = 0;
        });
      }
    },
    credit_sales_due_date(value) {
      this.invoice_doc.due_date = frappe.datetime.get_datetime_as_string(value)
      console.log(this.invoice_doc)
    },
    is_write_off_change(value) {
      if (value == 1) {
        this.invoice_doc.write_off_amount = this.diff_payment;
        this.invoice_doc.write_off_outstanding_amount_automatically = 1;
      } else {
        this.invoice_doc.write_off_amount = 0;
        this.invoice_doc.write_off_outstanding_amount_automatically = 0;
      }
    },
    redeemed_customer_credit(value) {
      if (value > this.available_customer_credit) {
        this.eventBus.emit("show_message", {
          title: `You can redeem customer credit upto ${this.available_customer_credit}`,
          color: "error",
        });
      }
    },
    sales_person() {
      if (this.sales_person) {
        this.invoice_doc.sales_team = [
          {
            sales_person: this.sales_person,
            allocated_percentage: 100,
          },
        ];
      } else {
        this.invoice_doc.sales_team = [];
      }
    },
  },
};
</script>

<style scoped>
.v-toolbar {
  height: 48px !important;
}

.v-toolbar__title {
  font-size: 1rem !important;
}

.v-text-field--dense .v-input__control {
  min-height: 32px !important;
}

.v-text-field--dense input {
  padding: 4px 0 !important;
}

.v-btn--size-small {
  height: 28px !important;
  font-size: 0.75rem !important;
  padding: 0 8px !important;
}

.v-btn--size-default {
  height: 36px !important;
  font-size: 0.875rem !important;
}

.v-btn.v-size--large {
  height: 44px !important;
  font-size: 1rem !important;
  padding: 0 16px !important;
}

.v-switch--dense {
  margin-top: 0 !important;
  padding-top: 0 !important;
}

.v-switch--dense .v-input__slot {
  margin-bottom: 0 !important;
}

.text-caption {
  font-size: 0.75rem !important;
  line-height: 1.2 !important;
}

.text-body-2 {
  font-size: 0.875rem !important;
  line-height: 1.25 !important;
}

.text-body-1 {
  font-size: 1rem !important;
}

.text-h6 {
  font-size: 1.125rem !important;
  line-height: 1.375 !important;
}

.v-chip--x-small {
  font-size: 10px !important;
  height: 16px !important;
}

[dir="rtl"] .v-icon {
  transform: scaleX(-1);
}

[dir="rtl"] .ml-1 {
  margin-left: 0 !important;
  margin-right: 0.25rem !important;
}

[dir="rtl"] .ml-2 {
  margin-left: 0 !important;
  margin-right: 0.5rem !important;
}

[dir="rtl"] .mr-2 {
  margin-right: 0 !important;
  margin-left: 0.5rem !important;
}

[dir="rtl"] .pr-1 {
  padding-right: 0 !important;
  padding-left: 4px !important;
}

[dir="rtl"] .pl-1 {
  padding-left: 0 !important;
  padding-right: 4px !important;
}

[dir="rtl"] .text-left {
  text-align: right !important;
}

[dir="rtl"] .text-right {
  text-align: left !important;
}

.v-container {
  max-width: 100%;
}

.v-row {
  margin: 0;
}

.v-col {
  padding: 2px;
}

.bg-primary-darken-1 {
  background-color: #1565C0 !important;
}

.text-white-70 {
  color: rgba(255, 255, 255, 0.7);
}

.v-divider {
  margin: 4px 0 !important;
}

.v-list-item {
  min-height: 36px !important;
  padding: 4px 12px !important;
}

.v-list-item__title {
  font-size: 0.75rem !important;
  line-height: 1.2 !important;
}

.v-list-item__subtitle {
  font-size: 0.625rem !important;
  line-height: 1.1 !important;
}
</style>