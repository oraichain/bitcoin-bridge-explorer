<script lang="ts" setup>
import { useBlockchain, useFormatter, useTxDialog } from '@/stores';
import { useWasmStore } from './WasmStore';
import { onMounted, ref } from 'vue';
import type { PaginabledCodeInfos } from './types';
import { PageRequest } from '@/types';
import PaginationBar from '@/components/PaginationBar.vue';
import router from '@/router';
import { toBase64, toHex } from '@cosmjs/encoding';
import type { QueryCodesResponse } from 'cosmjs-types/cosmwasm/wasm/v1/query';
import { accessTypeToJSON } from 'cosmjs-types/cosmwasm/wasm/v1/types';

const props = defineProps(['chain']);

const codes = ref({} as QueryCodesResponse | undefined);

const pageRequest = ref(new PageRequest());
const wasmStore = useWasmStore();
const dialog = useTxDialog();
const creator = ref('');
const field = ref('contract');
const history = ref([]);

function pageload(pageNum: number, nextKey?: Uint8Array) {
  pageRequest.value.setNextKey(nextKey);
  pageRequest.value.setPage(pageNum);

  wasmStore.wasmClient.getWasmCodeList(pageRequest.value).then((x) => {
    codes.value = x;
  });
}
pageload(1);

onMounted(() => {
  const historyStore = JSON.parse(
    localStorage.getItem('contract_history') || '{}'
  );
  history.value = historyStore[props.chain] || [];
});

function myContracts() {
  if (field.value === 'contract')
    router.push(
      `/${props.chain}/cosmwasm/0/transactions?contract=${creator.value}`
    );
  else if (field.value === 'creator')
    router.push(`/${props.chain}/cosmwasm/${creator.value}/contracts`);
}
const togo = ref('');
function gotoHistory() {
  router.push(`/${props.chain}/cosmwasm/0/transactions?contract=${togo.value}`);
}
</script>
<template>
  <div class="bg-base-100 px-4 pt-3 pb-4 rounded mb-4 shadow">
    <h2 class="card-title truncate w-full mb-4">{{ $t('cosmwasm.title') }}</h2>
    <div class="grid grid-flow-col auto-cols-max gap-4 overflow-hidden">
      <div class="join w-full border border-primary">
        <select v-model="field" class="select select-primary">
          <option value="contract">Contract</option>
          <option value="creator">Creator</option>
        </select>
        <input
          v-model="creator"
          type="text"
          class="input input-bordered w-full join-item"
          placeholder="address"
        />
        <button class="join-item btn btn-primary" @click="myContracts()">
          {{ $t('cosmwasm.btn_query') }}
        </button>
      </div>
      <div>
        <select
          v-model="togo"
          class="select select-primary"
          @change="gotoHistory()"
        >
          <option value="">History</option>
          <option v-for="(v, index) in history" :key="index" :value="v">
            ...{{ String(v).substring(45) }}
          </option>
        </select>
      </div>
    </div>

    <div class="overflow-x-auto">
      <table class="table table-compact w-full mt-4 text-sm">
        <thead class="bg-base-200">
          <tr>
            <th>{{ $t('cosmwasm.code_id') }}</th>
            <th>{{ $t('cosmwasm.code_hash') }}</th>
            <th>{{ $t('cosmwasm.creator') }}</th>
            <th>{{ $t('cosmwasm.permissions') }}</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="(v, index) in codes?.codeInfos" :key="index">
            <td>{{ v.codeId }}</td>
            <td>
              <RouterLink
                :to="`/${props.chain}/cosmwasm/${v.codeId}/contracts`"
                class="truncate max-w-[200px] block text-primary dark:invert"
                :title="v.dataHash"
              >
                {{ v.dataHash }}
              </RouterLink>
            </td>
            <td>{{ v.creator }}</td>
            <td>
              {{ v.instantiatePermission?.permission }}
              <span
                >{{ v.instantiatePermission?.address }}
                {{ v.instantiatePermission?.addresses.join(', ') }}</span
              >
            </td>
          </tr>
        </tbody>
      </table>
      <div class="flex justify-between">
        <PaginationBar
          :limit="pageRequest.limit"
          :total="
            codes?.pagination?.total ? codes.pagination.total.toString() : '0'
          "
          :nextKey="codes?.pagination?.nextKey"
          :callback="pageload"
        />
        <label
          for="wasm_store_code"
          class="btn btn-primary my-5"
          @click="dialog.open('wasm_store_code', {})"
          >{{ $t('cosmwasm.btn_up_sc') }}</label
        >
      </div>
    </div>
    <div class="w-[1px] h-[35px] bg-[#383B40] mx-2 hidden md:block"></div>
    <label
      for="wasm_store_code"
      class="btn btn-primary my-5 capitalize rounded-lg"
      @click="dialog.open('wasm_store_code', {})"
      >{{ $t('cosmwasm.btn_up_sc') }}</label
    >
  </div>
  <!-- <div class="overflow-x-auto">
      <table class="table table-compact w-full mt-4 text-sm text-white">
        <thead>
          <tr class="text-white">
            <th>{{ $t('cosmwasm.code_id') }}</th>
            <th>{{ $t('cosmwasm.code_hash') }}</th>
            <th>{{ $t('cosmwasm.creator') }}</th>
            <th class="text-right">{{ $t('cosmwasm.permissions') }}</th>
          </tr>
        </thead>
        <tbody>
          <tr
            v-for="(v, index) in codes?.codeInfos"
            :key="index"
            class="border-b border-b-[#242627] px-4"
          >
            <td>{{ v.codeId }}</td>
            <td>
              <RouterLink
                :to="`/${props.chain}/cosmwasm/${v.codeId}/contracts`"
                class="truncate max-w-[200px] block dark:text-[#B999F3]"
                :title="toHex(v.dataHash)"
              >
                {{ toHex(v.dataHash) }}
              </RouterLink>
            </td>
            <td>{{ v.creator }}</td>
            <td class="text-right">
              {{
                accessTypeToJSON(v.instantiatePermission?.permission)
                  .toLowerCase()
                  .replace(/^access_type/, '')
                  .replaceAll(/_(.)/g, (m, g) => ' ' + g.toUpperCase())
                  .trim()
              }}
              <span
                >{{ v.instantiatePermission?.address }}
                {{ v.instantiatePermission?.addresses.join(', ') }}</span
              >
            </td>
          </tr>
        </tbody>
      </table>
      <div class="flex justify-end">
        <PaginationBar
          :limit="pageRequest.limit"
          :total="
            codes?.pagination?.total ? codes.pagination.total.toString() : '0'
          "
          :nextKey="codes?.pagination?.nextKey"
          :callback="pageload"
        />
      </div>
    </div> -->
</template>

<route>
    {
      meta: {
        i18n: 'cosmwasm'
      }
    }
</route>
