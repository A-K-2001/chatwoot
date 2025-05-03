<script>
import { mapGetters } from 'vuex';
import axios from 'axios';
import ConversationHeader from './ConversationHeader.vue';
import DashboardAppFrame from '../DashboardApp/Frame.vue';
import EmptyState from './EmptyState/EmptyState.vue';
import MessagesView from './MessagesView.vue';
import ConversationSidebar from './ConversationSidebar.vue';

export default {
  components: {
    ConversationSidebar,
    ConversationHeader,
    DashboardAppFrame,
    EmptyState,
    MessagesView,
  },

  props: {
    inboxId: {
      type: [Number, String],
      default: '',
      required: false,
    },
    isInboxView: {
      type: Boolean,
      default: false,
    },
    isContactPanelOpen: {
      type: Boolean,
      default: true,
    },
    isOnExpandedLayout: {
      type: Boolean,
      default: true,
    },
  },
  emits: ['contactPanelToggle'],
  data() {
    return {
      activeIndex: 0,
      orderId: '',
      orderDetails: null,
      isLoading: false,
      searchError: null,
    };
  },
  computed: {
    ...mapGetters({
      currentChat: 'getSelectedChat',
      dashboardApps: 'dashboardApps/getRecords',
    }),
    dashboardAppTabs() {
      return [
        {
          key: 'messages',
          index: 0,
          name: this.$t('CONVERSATION.DASHBOARD_APP_TAB_MESSAGES'),
        },
        ...this.dashboardApps.map((dashboardApp, index) => ({
          key: `dashboard-${dashboardApp.id}`,
          index: index + 1,
          name: dashboardApp.title,
        })),
      ];
    },
    customTabs() {
      return [
        {
          key: 'customer-chat',
          index: 0,
          name: this.$t('CONVERSATION.CUSTOMER_CHAT_TAB'),
        },
        {
          key: 'order-details',
          index: 1,
          name: this.$t('CONVERSATION.ORDER_DETAILS_TAB'),
        },
      ];
    },
    showContactPanel() {
      return this.isContactPanelOpen && this.currentChat.id;
    },
  },
  watch: {
    'currentChat.inbox_id': {
      immediate: true,
      handler(inboxId) {
        if (inboxId) {
          this.$store.dispatch('inboxAssignableAgents/fetch', [inboxId]);
        }
      },
    },
    'currentChat.id'() {
      this.fetchLabels();
      this.activeIndex = 0;
    },
  },
  mounted() {
    this.fetchLabels();
    this.$store.dispatch('dashboardApps/get');
  },
  methods: {
    fetchLabels() {
      if (!this.currentChat.id) {
        return;
      }
      this.$store.dispatch('conversationLabels/get', this.currentChat.id);
    },
    onToggleContactPanel() {
      this.$emit('contactPanelToggle');
    },
    onDashboardAppTabChange(index) {
      this.activeIndex = index;
    },
    async searchOrder() {
      if (!this.orderId) {
        return;
      }

      this.isLoading = true;
      this.orderDetails = null;
      this.searchError = null;

      try {
        const response = await axios.get(
          `https://dev.2dopros.com/api/v1/admin/orders/${this.orderId}`,
          {
            headers: {
              Authorization:
                'Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ0b2tlblBheWxvYWQiOnsidXNlcl9pZCI6ImRjYTcwNTE2LTA0MGUtNGI2My05ZWJiLTk2YWJlYmRlZGRkYSIsInJvbGUiOiJBRE1JTiIsIm1vYmlsZSI6Iis5MTcyMzIwNzYzNjYifSwiaWF0IjoxNzQxNTExNTIyLCJleHAiOjE3NzMwNDc1MjJ9.k107GmyqZvrYz8aKjhjq0ZBjT4Rg2CqWDclHMKyTUWo',
            },
          }
        );

        this.orderDetails = response.data;
      } catch (error) {
        this.searchError =
          error.response?.data?.message || 'Error fetching order details';
        // eslint-disable-next-line no-console
        console.error('Error fetching order details:', error);
      } finally {
        this.isLoading = false;
      }
    },
  },
};
</script>

<template>
  <div
    class="conversation-details-wrap bg-n-background"
    :class="{
      'border-l rtl:border-l-0 rtl:border-r border-n-weak': !isOnExpandedLayout,
    }"
  >
    <ConversationHeader
      v-if="currentChat.id"
      :chat="currentChat"
      :is-inbox-view="isInboxView"
      :is-contact-panel-open="isContactPanelOpen"
      :show-back-button="isOnExpandedLayout && !isInboxView"
      @contact-panel-toggle="onToggleContactPanel"
    />
    <woot-tabs
      v-if="currentChat.id"
      :index="activeIndex"
      class="-mt-px bg-white dashboard-app--tabs dark:bg-slate-900"
      @change="onDashboardAppTabChange"
    >
      <woot-tabs-item
        v-for="tab in customTabs"
        :key="tab.key"
        :index="tab.index"
        :name="tab.name"
        :show-badge="false"
      />
    </woot-tabs>
    <div v-show="activeIndex === 0" class="flex h-full min-h-0 m-0">
      <MessagesView
        v-if="currentChat.id"
        :inbox-id="inboxId"
        :is-inbox-view="isInboxView"
        :is-contact-panel-open="isContactPanelOpen"
        @contact-panel-toggle="onToggleContactPanel"
      />
      <EmptyState
        v-if="!currentChat.id && !isInboxView"
        :is-on-expanded-layout="isOnExpandedLayout"
      />
      <ConversationSidebar
        v-if="showContactPanel"
        :current-chat="currentChat"
        @toggle-contact-panel="onToggleContactPanel"
      />
    </div>
    <div
      v-show="activeIndex === 1"
      class="flex flex-col h-full min-h-0 m-0 p-4 overflow-auto"
    >
      <div class="order-search-container w-full max-w-2xl mx-auto mb-6">
        <h3 class="text-lg font-medium mb-4">
          {{ $t('ORDER_SEARCH.SEARCH_PLACEHOLDER') }}
        </h3>
        <div class="flex gap-2">
          <input
            v-model="orderId"
            type="text"
            :placeholder="$t('ORDER_SEARCH.SEARCH_PLACEHOLDER')"
            class="flex-grow p-2 border border-black-300 rounded-md"
            @keyup.enter="searchOrder"
          />
          <button
            class="px-4 py-2 bg-woot-500 text-white rounded-md hover:bg-woot-600"
            :disabled="isLoading"
            @click="searchOrder"
          >
            {{
              isLoading
                ? $t('ORDER_SEARCH.LOADING')
                : $t('ORDER_SEARCH.SEARCH_BUTTON')
            }}
          </button>
        </div>
      </div>

      <div v-if="isLoading" class="text-center py-8">
        <div
          class="inline-block animate-spin rounded-full h-8 w-8 border-t-2 border-b-2 border-woot-500"
        />
        <p class="mt-2">{{ $t('ORDER_SEARCH.LOADING') }}</p>
      </div>

      <div v-else-if="searchError" class="text-center text-red-500 py-8">
        {{ searchError }}
      </div>

      <div v-else-if="!orderDetails && !isLoading" class="text-center py-8">
        <p>{{ $t('ORDER_SEARCH.EMPTY_STATE') }}</p>
      </div>

      <div
        v-else-if="orderDetails"
        class="order-details-container bg-white rounded-lg shadow-sm p-6 max-w-5xl mx-auto"
      >
        <!-- Header with order ID and status -->
        <div
          class="flex flex-col md:flex-row justify-between items-start md:items-center mb-6 pb-4 border-b border-black-200"
        >
          <h2 class="text-xl font-bold !text-black-900">
            Order #{{ orderDetails.order_id }}
          </h2>
          <div
            class="px-3 py-1 mt-2 md:mt-0 rounded-full text-sm font-medium"
            :class="{
              'bg-green-100 text-green-800': orderDetails.status === 'COMPLETE',
              'bg-blue-100 text-blue-800':
                orderDetails.status === 'IN_PROGRESS',
              'bg-yellow-100 text-yellow-800':
                orderDetails.status === 'PENDING',
              'bg-red-100 text-red-800': orderDetails.status === 'CANCELLED',
            }"
          >
            {{ orderDetails.status }}
          </div>
        </div>

        <!-- Dates Info -->
        <div class="mb-6 pb-4 border-b border-black-200">
          <h3 class="font-semibold text-black-900 mb-3">
            {{ $t('ORDER_SEARCH.DATES') }}
          </h3>
          <div class="space-y-1 text-sm !text-black-900">
            <div class="flex">
              <span class="text-black-500 w-32">{{ $t('ORDER_SEARCH.CREATED') }}:</span>
              {{ new Date(orderDetails.created_at).toLocaleString() }}
            </div>
            <div class="flex">
              <span class="text-black-500 w-32">{{ $t('ORDER_SEARCH.EXPECTED_START') }}:</span>
              {{ new Date(orderDetails.expected_started_at).toLocaleString() }}
            </div>
            <div class="flex">
              <span class="text-black-500 w-32">{{ $t('ORDER_SEARCH.EXPECTED_END') }}:</span>
              {{ new Date(orderDetails.expected_ended_at).toLocaleString() }}
            </div>
            <div v-if="orderDetails.started_at" class="flex">
              <span class="text-black-500 w-32">{{ $t('ORDER_SEARCH.ACTUAL_START') }}:</span>
              {{ new Date(orderDetails.started_at).toLocaleString() }}
            </div>
            <div v-if="orderDetails.ended_at" class="flex">
              <span class="text-black-500 w-32">{{ $t('ORDER_SEARCH.ACTUAL_END') }}:</span>
              {{ new Date(orderDetails.ended_at).toLocaleString() }}
            </div>
          </div>
        </div>

        <!-- Time and Cost -->
        <div class="mb-6 pb-4 border-b border-black-200">
          <h3 class="font-semibold text-black-900 mb-3">
            {{ $t('ORDER_SEARCH.TIME_AND_COST') }}
          </h3>
          <div class="space-y-1 text-sm !text-black-900">
            <div class="flex">
              <span class="text-black-500 w-32">{{ $t('ORDER_SEARCH.EXPECTED_TIME') }}:</span>
              {{ orderDetails.expected_time_taken }}
              {{ $t('ORDER_SEARCH.MINUTES') }}
            </div>
            <div v-if="orderDetails.time_taken" class="flex">
              <span class="text-black-500 w-32">{{ $t('ORDER_SEARCH.ACTUAL_TIME') }}:</span>
              {{ orderDetails.time_taken }} {{ $t('ORDER_SEARCH.MINUTES') }}
            </div>
            <div class="flex">
              <span class="text-black-500 w-32">{{ $t('ORDER_SEARCH.EXPECTED_AMOUNT') }}:</span>
              ${{ (orderDetails.expected_amount / 100).toFixed(2) }}
            </div>
            <div class="flex">
              <span class="text-black-500 w-32">{{ $t('ORDER_SEARCH.ACTUAL_AMOUNT') }}:</span>
              ${{ (orderDetails.amount / 100).toFixed(2) }}
            </div>
          </div>
        </div>

        <!-- Customer Info -->
        <div class="mb-6 pb-4 border-b border-black-200">
          <h3 class="font-semibold text-black-900 mb-3">
            {{ $t('ORDER_SEARCH.CUSTOMER') }}
          </h3>
          <div class="space-y-1 text-sm !text-black-900">
            <div class="flex">
              <span class="text-black-500 w-32">{{ $t('ORDER_SEARCH.NAME') }}:</span>
              {{ orderDetails.user.name }}
            </div>
            <div class="flex">
              <span class="text-black-500 w-32">{{ $t('ORDER_SEARCH.EMAIL') }}:</span>
              {{ orderDetails.user.email }}
            </div>
            <div class="flex">
              <span class="text-black-500 w-32">{{ $t('ORDER_SEARCH.PHONE') }}:</span>
              {{ orderDetails.user.mobile }}
            </div>
            <div v-if="orderDetails.user.gender" class="flex">
              <span class="text-black-500 w-32">{{ $t('ORDER_SEARCH.GENDER') }}:</span>
              {{ orderDetails.user.gender }}
            </div>
          </div>
        </div>

        <!-- Service Info -->
        <div class="mb-6 pb-4 border-b border-black-200">
          <h3 class="font-semibold text-black-900 mb-3">
            {{ $t('ORDER_SEARCH.SERVICE') }}
          </h3>
          <div class="mb-3">
            <div class="font-medium text-black-900">
              {{ orderDetails.service.title }}
            </div>
            <div class="text-sm text-black-800">
              {{ orderDetails.service.subtitle }}
            </div>
            <div class="mt-2 flex justify-between">
              <span class="text-black-900">{{
                $t('ORDER_SEARCH.BASE_PRICE')
              }}</span>
              <span class="text-black-900">
                ${{ (orderDetails.service.amount / 100).toFixed(2) }}
              </span>
            </div>
          </div>
          <div
            class="text-sm !text-black-900 max-h-96 overflow-y-auto whitespace-pre-line"
          >
            {{ orderDetails.service.description }}
          </div>
        </div>

        <!-- Address Info -->
        <div class="mb-6 pb-4 border-b border-black-200">
          <h3 class="font-semibold text-black-900 mb-3">
            {{ $t('ORDER_SEARCH.ADDRESS') }}
          </h3>
          <div class="text-sm !text-black-900 font-medium">
            <p class="font-semibold text-black-900">
              {{ orderDetails.address.label }}
            </p>
            <p class="my-1 text-black-900 font-medium">
              {{ orderDetails.address.line1 }}
            </p>
            <p
              v-if="orderDetails.address.line2"
              class="my-1 text-black-900 font-medium"
            >
              {{ orderDetails.address.line2 }}
            </p>
            <p class="my-1 text-black-900 font-medium">
              {{ orderDetails.address.area }}, {{ orderDetails.address.city }} -
              {{ orderDetails.address.pincode }}
            </p>
            <div class="mt-2 text-black-900 font-medium">
              {{ $t('ORDER_SEARCH.COORDINATES') }}:
              {{ orderDetails.address.latitude }},
              {{ orderDetails.address.longitude }}
            </div>
          </div>
        </div>

        <!-- Charges Section -->
        <div
          v-if="orderDetails.charges && orderDetails.charges.length"
          class="mb-6 pb-4 border-b border-black-200"
        >
          <h3 class="font-semibold text-black-900 mb-3">
            {{ $t('ORDER_SEARCH.ADDITIONAL_CHARGES') }}
          </h3>
          <div class="overflow-x-auto">
            <table class="w-full text-sm !text-black-900 font-medium">
              <thead>
                <tr class="border-b border-black-200">
                  <th class="py-2 pr-4 text-left font-semibold text-black-900">
                    {{ $t('ORDER_SEARCH.TYPE') }}
                  </th>
                  <th class="py-2 pr-4 text-left font-semibold text-black-900">
                    {{ $t('ORDER_SEARCH.AMOUNT') }}
                  </th>
                  <th class="py-2 pr-4 text-left font-semibold text-black-900">
                    {{ $t('ORDER_SEARCH.REMARK') }}
                  </th>
                  <th class="py-2 pr-4 text-left font-semibold text-black-900">
                    {{ $t('ORDER_SEARCH.STATUS') }}
                  </th>
                  <th class="py-2 pr-4 text-left font-semibold text-black-900">
                    {{ $t('ORDER_SEARCH.DATE') }}
                  </th>
                </tr>
              </thead>
              <tbody>
                <tr
                  v-for="charge in orderDetails.charges"
                  :key="charge.id"
                  class="border-b border-black-100"
                >
                  <td class="py-2 pr-4 text-black-900 font-medium">
                    {{ charge.charge_type }}
                  </td>
                  <td class="py-2 pr-4 text-black-900 font-medium">
                    ${{ (charge.amount / 100).toFixed(2) }}
                  </td>
                  <td class="py-2 pr-4 text-black-900 font-medium">
                    {{ charge.remark || '-' }}
                  </td>
                  <td class="py-2 pr-4">
                    <span
                      class="px-2 py-0.5 rounded-full text-xs"
                      :class="{
                        'bg-green-100 text-green-800':
                          charge.status === 'ACCEPTED',
                        'bg-red-100 text-red-800': charge.status === 'REJECTED',
                        'bg-yellow-100 text-yellow-800':
                          charge.status === 'PENDING',
                      }"
                    >
                      {{ charge.status }}
                    </span>
                  </td>
                  <td class="py-2 pr-4 text-black-900 font-medium">
                    {{ new Date(charge.created_at).toLocaleString() }}
                  </td>
                </tr>
              </tbody>
              <tfoot>
                <tr class="font-semibold">
                  <td class="py-2 pr-4 font-semibold text-black-900">
                    {{ $t('ORDER_SEARCH.TOTAL') }}
                  </td>
                  <td class="py-2 pr-4 font-semibold text-black-900">
                    ${{
                      (
                        orderDetails.charges.reduce(
                          (sum, charge) =>
                            sum +
                            (charge.status === 'ACCEPTED' ? charge.amount : 0),
                          0
                        ) / 100
                      ).toFixed(2)
                    }}
                  </td>
                  <td colspan="3" />
                </tr>
              </tfoot>
            </table>
          </div>
        </div>

        <!-- Quotations Section -->
        <div
          v-if="orderDetails.qotation && orderDetails.qotation.length"
          class="mb-6 pb-4 border-b border-black-200"
        >
          <h3 class="font-semibold text-black-900 mb-3">
            {{ $t('ORDER_SEARCH.QUOTATIONS') || 'Quotations' }}
          </h3>
          <div class="space-y-4">
            <div
              v-for="quote in orderDetails.qotation"
              :key="quote.qotation_id"
              class="bg-black-50 rounded-lg p-4 border border-black-200"
            >
              <div
                class="flex flex-col md:flex-row justify-between items-start md:items-center mb-2"
              >
                <div class="font-bold text-black-900">
                  Quotation #{{ quote.qotation_id.substring(0, 8) }}
                </div>
                <div
                  class="px-3 py-1 mt-2 md:mt-0 rounded-full text-sm font-semibold"
                  :class="{
                    'bg-green-100 text-green-800': quote.status === 'ACCEPTED',
                    'bg-red-100 text-red-800': quote.status === 'REJECTED',
                    'bg-yellow-100 text-yellow-800': quote.status === 'PENDING',
                  }"
                >
                  {{ quote.status }}
                </div>
              </div>

              <div class="space-y-2 mt-3">
                <div class="flex">
                  <span class="text-black-800 w-32 font-semibold">Amount:</span>
                  <span class="text-black-900 font-semibold">${{ (quote.amount / 100).toFixed(2) }}</span>
                </div>
                <div class="flex">
                  <span class="text-black-800 w-32 font-semibold">Created:</span>
                  <span class="text-black-900 font-semibold">{{
                    new Date(quote.created_at).toLocaleString()
                  }}</span>
                </div>
                <div v-if="quote.details" class="mt-2">
                  <span class="text-black-800 w-32 font-semibold block mb-1">Details:</span>
                  <div
                    class="text-black-900 font-semibold whitespace-pre-line pl-6"
                  >
                    {{ quote.details }}
                  </div>
                </div>
                <div v-if="quote.user_remark" class="mt-2">
                  <span class="text-black-800 w-32 font-semibold block mb-1">User Remark:</span>
                  <div
                    class="text-black-900 font-semibold whitespace-pre-line pl-6"
                  >
                    {{ quote.user_remark }}
                  </div>
                </div>
                <div v-if="quote.images && quote.images.length" class="mt-4">
                  <span class="text-black-800 w-32 font-semibold block mb-2">Images:</span>
                  <div class="flex flex-wrap gap-2">
                    <img
                      v-for="(image, index) in quote.images"
                      :key="index"
                      :src="image"
                      class="w-24 h-24 object-cover rounded-md border border-black-300 shadow-sm"
                      alt="Quotation image"
                      @click="window.open(image, '_blank')"
                    />
                  </div>
                </div>
              </div>
            </div>
          </div>
        </div>

        <!-- Worker/Contractor Grid -->
        <div class="grid grid-cols-1 md:grid-cols-2 gap-6">
          <!-- Worker Info -->
          <div v-if="orderDetails.worker" class="mb-6">
            <h3 class="font-semibold text-black-900 mb-3">
              {{ $t('ORDER_SEARCH.WORKER') }}
            </h3>
            <div class="space-y-1 text-sm !text-black-900">
              <div class="flex">
                <span class="text-black-500 w-32">{{ $t('ORDER_SEARCH.NAME') }}:</span>
                {{ orderDetails.worker.user.name }}
              </div>
              <div class="flex">
                <span class="text-black-500 w-32">{{ $t('ORDER_SEARCH.EMAIL') }}:</span>
                {{ orderDetails.worker.user.email }}
              </div>
              <div class="flex">
                <span class="text-black-500 w-32">{{ $t('ORDER_SEARCH.PHONE') }}:</span>
                {{ orderDetails.worker.user.mobile }}
              </div>
              <div class="flex">
                <span class="text-black-500 w-32">{{ $t('ORDER_SEARCH.ACTIVE') }}:</span>
                {{
                  orderDetails.worker.active
                    ? $t('ORDER_SEARCH.YES')
                    : $t('ORDER_SEARCH.NO')
                }}
              </div>
            </div>
          </div>

          <!-- Contractor Info -->
          <div v-if="orderDetails.contractor" class="mb-6">
            <h3 class="font-semibold text-black-900 mb-3">
              {{ $t('ORDER_SEARCH.CONTRACTOR') }}
            </h3>
            <div class="space-y-1 text-sm !text-black-900">
              <div class="flex">
                <span class="text-black-500 w-32">{{ $t('ORDER_SEARCH.NAME') }}:</span>
                {{ orderDetails.contractor.user.name }}
              </div>
              <div class="flex">
                <span class="text-black-500 w-32">{{ $t('ORDER_SEARCH.EMAIL') }}:</span>
                {{ orderDetails.contractor.user.email }}
              </div>
              <div class="flex">
                <span class="text-black-500 w-32">{{ $t('ORDER_SEARCH.PHONE') }}:</span>
                {{ orderDetails.contractor.user.mobile }}
              </div>
              <div class="flex">
                <span class="text-black-500 w-32">{{ $t('ORDER_SEARCH.COMMISSION') }}:</span>
                {{ orderDetails.contractor.commission_percentage }}%
              </div>
            </div>
          </div>
        </div>

        <!-- Remarks Section -->
        <div
          v-if="orderDetails.user_remark || orderDetails.worker_remark"
          class="mt-3"
        >
          <h3 class="font-semibold text-black-900 mb-3">
            {{ $t('ORDER_SEARCH.REMARKS') }}
          </h3>
          <div v-if="orderDetails.user_remark" class="mb-4">
            <div class="text-sm text-black-900 mb-1 font-medium">
              {{ $t('ORDER_SEARCH.USER_REMARK') }}:
            </div>
            <p class="whitespace-pre-line text-sm text-black-900">
              {{ orderDetails.user_remark }}
            </p>
          </div>
          <div v-if="orderDetails.worker_remark">
            <div class="text-sm text-black-900 mb-1 font-medium">
              {{ $t('ORDER_SEARCH.WORKER_REMARK') }}:
            </div>
            <p class="whitespace-pre-line text-sm text-black-900">
              {{ orderDetails.worker_remark }}
            </p>
          </div>
        </div>
      </div>
    </div>
    <DashboardAppFrame
      v-for="(dashboardApp, index) in dashboardApps"
      v-show="activeIndex - 2 === index"
      :key="currentChat.id + '-' + dashboardApp.id"
      :is-visible="activeIndex - 2 === index"
      :config="dashboardApps[index].content"
      :position="index"
      :current-chat="currentChat"
    />
  </div>
</template>

<style lang="scss" scoped>
.conversation-details-wrap {
  @apply flex flex-col min-w-0 w-full;
}

.dashboard-app--tabs {
  ::v-deep {
    .tabs-title {
      a {
        @apply pb-2 pt-1;
      }
    }
  }
}
</style>
