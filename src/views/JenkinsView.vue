<template>
  <div class="jobs-list">
    <n-tabs
      class="card-tabs"
      default-value="favorites"
      size="large"
      animated
      @update:value="handleTabChange"
      pane-style="padding-top: 15px; box-sizing: border-box;">
      <n-tab-pane name="favorites" tab="关注">
        <favorite-jobs 
          :active="activeTab === 'favorites'"
          @build="handleFavoriteBuild" 
          ref="favoriteJobsRef"
        />
      </n-tab-pane>
      <n-tab-pane name="jobs" tab="任务">
        <jenkins ref="jenkinsRef"/>
      </n-tab-pane>
      <n-tab-pane name="config" tab="配置">
        <config/>
      </n-tab-pane>
    </n-tabs>
  </div>
</template>

<script lang="ts">
import { Options, Vue } from "vue-class-component";
import Jenkins from "@/components/Jenkins.vue"; // @ is an alias to /src
import Config from "@/components/Config.vue";
import FavoriteJobs from "@/components/FavoriteJobs.vue";
import { useStore } from 'vuex';

@Options({
  components: {
    Jenkins,
    Config,
    FavoriteJobs
  },
})
export default class HomeView extends Vue {
  activeTab = 'favorites';
  store = useStore();

  handleFavoriteBuild(job: any) {
    const jenkinsRef = this.$refs.jenkinsRef as any;
    if (jenkinsRef) {
      jenkinsRef.buildJobParamsDialog(job, 0);
    }
  }

  handleTabChange(value: string) {
    this.activeTab = value;
    if (value === 'favorites') {
      const favoriteJobsRef = this.$refs.favoriteJobsRef as any;
      if (favoriteJobsRef) {
        favoriteJobsRef.loadFavorites();
      }
    } else if (value === 'jobs') {
      const jenkinsRef = this.$refs.jenkinsRef as any;
      if (jenkinsRef) {
        // 获取当前配置的 Jenkins 实例
        const currentConfig = this.store.state.config;
        // 使用配置的默认视图，如果没有则使用 'all'
        const defaultView = currentConfig?.defaultView || 'all';
        jenkinsRef.viewName = defaultView;
        jenkinsRef.loadJobs();
      }
    }
  }
}
</script>

<style scoped>
  .n-tabs .n-tab-pane {
    padding: 0;
  }

  .card-tabs .n-tabs-nav--bar-type {
    padding-left: 4px;
  }

  .jobs-list {
    padding: 0 10px;
  }

  .jenkins-view {
    padding: 16px;
  }

  :deep(.n-tabs-nav) {
    padding-left: 16px;
  }

  :deep(.n-tabs-content) {
    padding: 0 10px !important;
  }
</style>
