<template>
  <Layout>
    <template #content-left>
      <div class="px-4 px-md-0 mb-3">
        <router-link
          :to="{ name: domain ? 'home' : 'proposals' }"
          class="text-gray"
        >
          <Icon name="back" size="22" class="v-align-middle" />
          {{ space.name }}
        </router-link>
      </div>
      <div class="px-4 px-md-0">
        <template v-if="loaded">
          <h1 class="mb-2">
            {{ payload.name }}
            <span v-text="`#${id.slice(0, 7)}`" class="text-gray" />
          </h1>
          <div class="mb-4">
            <State :proposal="proposal" />
            <UiDropdown
              top="2.2rem"
              right="1.3rem"
              class="float-right"
              v-if="isAdmin || isCreator"
              @select="selectFromDropdown"
              :items="[{ text: $t('deleteProposal'), action: 'delete' }]"
            >
              <div class="pr-3">
                <UiLoading v-if="dropdownLoading" />
                <Icon
                  v-else
                  name="threedots"
                  size="25"
                  class="v-align-text-bottom"
                />
              </div>
            </UiDropdown>
          </div>
          <UiMarkdown :body="payload.body" class="mb-6" />
        </template>
        <PageLoading v-else />
      </div>
      <Block
        v-if="loaded && ts >= payload.start && ts < payload.end"
        class="mb-4"
        :title="$t('proposal.castVote')"
      >
        <div class="mb-3">
          <UiButton
            v-for="(choice, i) in payload.choices"
            :key="i"
            @click="selectedChoice = i + 1"
            class="d-block width-full mb-2"
            :class="selectedChoice === i + 1 && 'button--active'"
          >
            {{ _shorten(choice, 32) }}
            <a
              v-if="payload.metadata.plugins?.aragon?.[`choice${[i + 1]}`]"
              @click="modalOpen = true"
              :aria-label="`Target address: ${
                payload.metadata.plugins.aragon[`choice${i + 1}`].actions[0].to
              }\nValue: ${
                payload.metadata.plugins.aragon[`choice${i + 1}`].actions[0]
                  .value
              }\nData: ${
                payload.metadata.plugins.aragon[`choice${i + 1}`].actions[0]
                  .data
              }`"
              class="tooltipped tooltipped-n break-word"
            >
              <Icon name="warning" class="v-align-middle ml-1" />
            </a>
          </UiButton>
        </div>
        <UiButton
          :disabled="voteLoading || app.authLoading || !selectedChoice"
          :loading="voteLoading"
          @click="web3.account ? (modalOpen = true) : (modalAccountOpen = true)"
          class="d-block width-full button--submit"
        >
          {{ $t('proposal.vote') }}
        </UiButton>
      </Block>
      <BlockVotes
        v-if="loaded"
        :loaded="loadedResults"
        :space="space"
        :proposal="proposal"
        :votes="votes"
      />
    </template>
    <template #sidebar-right v-if="loaded">
      <Block :title="$t('information')">
        <div class="mb-1">
          <b>{{ $t('strategies') }}</b>
          <span
            @click="modalStrategiesOpen = true"
            class="float-right text-white a"
          >
            <span v-for="(symbol, symbolIndex) of symbols" :key="symbol">
              <span :aria-label="symbol" class="tooltipped tooltipped-n">
                <Token :space="space.key" :symbolIndex="symbolIndex" />
              </span>
              <span v-show="symbolIndex !== symbols.length - 1" class="ml-1" />
            </span>
          </span>
        </div>
        <div class="mb-1">
          <b>{{ $t('author') }}</b>
          <User
            :address="proposal.address"
            :profile="proposal.profile"
            :space="space"
            class="float-right"
          />
        </div>
        <div class="mb-1">
          <b>IPFS</b>
          <a
            :href="_ipfsUrl(proposal.ipfsHash)"
            target="_blank"
            class="float-right"
          >
            #{{ proposal.ipfsHash.slice(0, 7) }}
            <Icon name="external-link" class="ml-1" />
          </a>
        </div>
        <div>
          <div class="mb-1">
            <b>{{ $t('proposal.startDate') }}</b>
            <span
              :aria-label="_ms(payload.start)"
              v-text="$d(payload.start * 1e3, 'short', 'en-US')"
              class="float-right text-white tooltipped tooltipped-n"
            />
          </div>
          <div class="mb-1">
            <b>{{ $t('proposal.endDate') }}</b>
            <span
              :aria-label="_ms(payload.end)"
              v-text="$d(payload.end * 1e3, 'short', 'en-US')"
              class="float-right text-white tooltipped tooltipped-n"
            />
          </div>
          <div class="mb-1">
            <b>{{ $t('snapshot') }}</b>
            <a
              :href="_explorer(space.network, payload.snapshot, 'block')"
              target="_blank"
              class="float-right"
            >
              {{ _n(payload.snapshot, '0,0') }}
              <Icon name="external-link" class="ml-1" />
            </a>
          </div>
        </div>
      </Block>
      <BlockResults
        :loaded="loadedResults"
        :id="id"
        :space="space"
        :payload="payload"
        :results="results"
        :votes="votes"
      />
      <div v-if="loadedResults">
        <PluginAragonCustomBlock
          :loaded="loadedResults"
          :id="id"
          :space="space"
          :payload="payload"
          :results="results"
        />
        <PluginGnosisCustomBlock
          v-if="payload.metadata.plugins?.gnosis?.baseTokenAddress"
          :proposalConfig="payload.metadata.plugins.gnosis"
          :choices="payload.choices"
        />
        <PluginDaoModuleCustomBlock
          v-if="payload.metadata.plugins?.daoModule?.txs"
          :proposalConfig="payload.metadata.plugins.daoModule"
          :proposalEnd="payload.end"
          :porposalId="id"
          :moduleAddress="space.plugins?.daoModule?.address"
          :network="space.network"
        />
        <PluginQuorumCustomBlock
          :loaded="loadedResults"
          v-if="space.plugins?.quorum"
          :space="space"
          :payload="payload"
          :results="results"
        />
      </div>
    </template>
  </Layout>
  <teleport to="#modal">
    <ModalConfirm
      v-if="loaded"
      :open="modalOpen"
      @close="modalOpen = false"
      @reload="loadProposal"
      :space="space"
      :proposal="proposal"
      :id="id"
      :selectedChoice="selectedChoice"
      :totalScore="totalScore"
      :scores="scores"
      :snapshot="payload.snapshot"
    />
    <ModalStrategies
      :open="modalStrategiesOpen"
      @close="modalStrategiesOpen = false"
      :space="space"
      :strategies="space.strategies"
    />
  </teleport>
</template>

<script>
import { mapActions } from 'vuex';
import { getProposal, getResults, getPower } from '@/helpers/snapshot';
import { useModal } from '@/composables/useModal';

export default {
  setup() {
    const { modalAccountOpen } = useModal();
    return { modalAccountOpen };
  },
  data() {
    return {
      key: this.$route.params.key,
      id: this.$route.params.id,
      loading: false,
      loaded: false,
      loadedResults: false,
      voteLoading: false,
      dropdownLoading: false,
      proposal: {},
      votes: {},
      results: [],
      modalOpen: false,
      modalStrategiesOpen: false,
      selectedChoice: 0,
      totalScore: 0,
      scores: []
    };
  },
  computed: {
    space() {
      return this.app.spaces[this.key];
    },
    payload() {
      return this.proposal.msg.payload;
    },
    ts() {
      return (Date.now() / 1e3).toFixed();
    },
    symbols() {
      return this.space.strategies.map(strategy => strategy.params.symbol);
    },
    isCreator() {
      return this.proposal.address === this.web3.account;
    },
    isAdmin() {
      const admins = (this.space.admins || []).map(admin =>
        admin.toLowerCase()
      );
      return admins.includes(this.web3.account?.toLowerCase());
    }
  },
  watch: {
    'web3.account': async function (val, prev) {
      if (val?.toLowerCase() !== prev) await this.loadPower();
    }
  },
  methods: {
    ...mapActions(['send']),
    async loadProposal() {
      const proposalObj = await getProposal(this.space, this.id);
      const { proposal, votes, blockNumber } = proposalObj;
      this.proposal = proposal;
      this.loaded = true;
      const resultsObj = await getResults(
        this.space,
        proposal,
        votes,
        blockNumber
      );
      this.votes = resultsObj.votes;
      this.results = resultsObj.results;
      this.loadedResults = true;
    },
    async loadPower() {
      if (!this.web3.account || !this.proposal.address) return;
      const { scores, totalScore } = await getPower(
        this.space,
        this.web3.account,
        this.payload.snapshot
      );
      this.totalScore = totalScore;
      this.scores = scores;
    },
    async deleteProposal() {
      this.dropdownLoading = true;
      try {
        if (
          await this.send({
            space: this.space.key,
            type: 'delete-proposal',
            payload: {
              proposal: this.id
            }
          })
        ) {
          this.dropdownLoading = false;
          this.$router.push({
            name: 'proposals'
          });
        }
      } catch (e) {
        console.error(e);
      }
      this.dropdownLoading = false;
    },
    selectFromDropdown(e) {
      if (e === 'delete') this.deleteProposal();
    }
  },
  async created() {
    this.loading = true;
    await this.loadProposal();
    this.loading = false;
    await this.loadPower();
  }
};
</script>
