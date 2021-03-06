<template>
  <div class="buy-subdomain-container">
    <back-button />
    <div class="buy-subdomain-content">
      <div class="buy-subdomain-form-container">
        <div class="title">
          <h4>{{ $t('subDomain.title') }}</h4>
        </div>
        <div class="form">
          <div class="subdomain-input">
            <input
              :placeholder="$t('subDomain.ph-enter-name')"
              :class="hasError ? 'errorInput' : ''"
              type="text"
              @input="debounceInput"
            />
            <button type="button" @click="query">
              {{ $t('subDomain.check') }}
            </button>
          </div>
          <p v-if="hasError" class="errorText">
            <span>{{ $t('subDomain.invalid-symbol') }}</span>
          </p>
        </div>
        <div v-show="results.length > 0" class="result-section">
          <div class="title">
            <h4>{{ $t('subDomain.all') }}</h4>
          </div>
          <div class="results-container">
            <div
              v-for="(item, index) in sortedResults"
              :key="domainName + item.domain + index"
              :class="[item.active ? '' : 'disabled', 'result-item']"
            >
              <span class="domain-name"
                >{{ domainName }}.{{ item.domain }}.eth</span
              >
              <div class="buy-button-container">
                <span class="amt"
                  >{{ web3.utils.fromWei(item.price, 'ether') }}
                  {{ $t('common.currency.eth') }}</span
                >
                <button @click="buyDomain(item)">
                  <span v-if="item.active">{{ $t('subDomain.buy') }}</span>
                  <span v-else>
                    <i class="fa fa-times" />
                  </span>
                </button>
              </div>
            </div>
          </div>
        </div>
      </div>
      <div>
        <interface-bottom-text
          :link-text="$t('common.help-center')"
          :question="$t('common.have-issues')"
          link="https://kb.myetherwallet.com"
        />
      </div>
    </div>
  </div>
</template>

<script>
import InterfaceBottomText from '@/components/InterfaceBottomText';
import BackButton from '@/layouts/InterfaceLayout/components/BackButton';
import SubdomainAbi from '@/helpers/subdomainAbi.js';
import domains from './domains.json';
import normalise from '@/helpers/normalise';
import BigNumber from 'bignumber.js';
import web3 from 'web3';
import { mapState } from 'vuex';
import { Toast } from '@/helpers';
export default {
  components: {
    'interface-bottom-text': InterfaceBottomText,
    'back-button': BackButton
  },
  data() {
    return {
      subdomainContract: function() {},
      ensContract: function() {},
      results: [],
      domainName: '',
      knownRegistrarInstances: {},
      hasError: false
    };
  },
  computed: {
    ...mapState(['ethDonationAddress', 'ens', 'account', 'web3']),
    sortedResults() {
      const newArr = this.results;
      newArr.sort((a, b) => {
        const ab = new BigNumber(a.price).gt(b.price)
          ? -1
          : new BigNumber(a.price).eq(b.price)
          ? 0
          : 1;
        return ab;
      });
      const taken = newArr.filter(item => {
        return item.active === false;
      });
      const available = newArr.filter(item => {
        return item.active === true;
      });
      return available.concat(taken);
    }
  },
  watch: {
    domainName() {
      this.query();
    }
  },
  mounted() {
    const web3C = this.web3.eth.Contract;
    domains.forEach(domain => {
      const updatedDomain = Object.assign({}, domain);
      updatedDomain.contract = new web3C(SubdomainAbi, domain.registrar);
      this.knownRegistrarInstances[domain.name] = updatedDomain;
    });
  },
  methods: {
    debounceInput: web3.utils._.debounce(function(e) {
      try {
        this.domainName = normalise(e.target.value);
        this.hasError = false;
      } catch (e) {
        Toast.responseHandler(e, Toast.WARN);
        this.hasError = true;
        return;
      }
    }, 1500),
    async query() {
      this.results = [];
      const sha3 = this.web3.utils.sha3;
      if (this.domainName.length > 1) {
        for (const key in this.knownRegistrarInstances) {
          const getSubdomain = await this.knownRegistrarInstances[
            key
          ].contract.methods
            .query(sha3(key), this.domainName)
            .call();
          getSubdomain.version = this.knownRegistrarInstances[key].version;
          if (getSubdomain[0] !== '') {
            getSubdomain.active = true;
          } else {
            getSubdomain.active = false;
            getSubdomain.domain = key;
          }
          this.results.push(getSubdomain);
        }
      }
    },
    async buyDomain(item) {
      const domain = this.web3.utils.sha3(item.domain);
      const subdomain = this.domainName;
      const ownerAddress = this.account.address;
      const referrerAddress = this.ethDonationAddress;
      const resolverAddress = await this.ens.resolver('resolver.eth').addr();
      const itemContract = this.knownRegistrarInstances[item.domain];
      const data = await (item.version === '1.0'
        ? itemContract.contract.methods
            .register(
              domain,
              subdomain,
              ownerAddress,
              referrerAddress,
              resolverAddress
            )
            .encodeABI()
        : itemContract.contract.methods
            .register(
              domain,
              subdomain,
              ownerAddress,
              resolverAddress,
              referrerAddress
            )
            .encodeABI());
      const raw = {
        from: ownerAddress,
        data: data,
        to: itemContract.registrar,
        value: item.price
      };
      this.web3.eth.sendTransaction(raw).catch(err => {
        Toast.responseHandler(err, false);
      });
    }
  }
};
</script>

<style lang="scss" scoped>
@import 'BuySubdomain.scss';
</style>
