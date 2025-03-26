<template>
  <div class="favorite-jobs">
    <n-card title="关注的项目" class="favorite-jobs-card">
      <n-empty v-if="!favoriteJobs.length" description="暂无关注的项目" />
      
      <n-data-table
        v-else
        :columns="columns"
        :data="favoriteJobs"
        :pagination="false"
        :bordered="false"
        :single-line="false"
        size="small"
      />
    </n-card>

    <!-- 获取构建参数弹出框 -->
    <n-modal v-model:show="showbuildParamsModal" 
        preset="dialog" 
        title="Dialog" 
        :close-on-esc="false"
        :mask-closable="false"
        :showIcon="false">
        <template #header>
            <div>构建({{currentBuildJob.name}})</div>
        </template>
        <n-space vertical>
          <n-spin :show="buildParamsLoading">
            <n-form-item label="描述" v-if="currentBuildJob.description">
              <label style="color: #a19f9d;">{{ currentBuildJob.description }}</label>
            </n-form-item>
            <n-form ref="buildJobRef" :model="currentBuildJob" v-if="currentBuildJob.parameterProcessed">
              <template v-for="params in currentBuildJob.parameterProcessed" v-bind:key="params.name">
                <n-form-item>
                  <template #label>
                     {{params.name}}&nbsp;
                     <n-tooltip trigger="hover" v-if="params.description">
                      <template #trigger>
                        <n-icon size="20" color="#a19f9d">
                          <help-filled />
                        </n-icon>
                      </template>
                      {{ params.description }}
                    </n-tooltip>
                  </template>
                  <n-input v-if="jobParameterTypeOfInput.indexOf(params.type) !== -1" 
                    :type="params.type === 'PasswordParameterDefinition' ? 'password' : 'text'"
                    v-model:value="currentBuildJob.form[params.name]"/>
                  <n-input v-else-if="jobParameterTypeOfTextArea.indexOf(params.type) !== -1" type="textarea"
                    v-model:value="currentBuildJob.form[params.name]"/>
                  <n-select v-else-if="jobParameterTypeOfSelect.indexOf(params.type) !== -1 && params.choices && params.choices.length !== 0"
                    v-model:value="currentBuildJob.form[params.name]" :multiple="params.type === 'PT_MULTI_SELECT'" :options="paramsSelectOptions" />
                  <n-switch v-else-if="jobParameterTypeOfCheckbox.indexOf(params.type) !== -1"
                     v-model:value="currentBuildJob.form[params.name]"/>
                  <label v-else>不支持的参数类型：{{ params.type }}</label>
                </n-form-item>
              </template>
            </n-form>
            <template #description>
              加载构建参数。。。
            </template>
          </n-spin>
        </n-space>
        <template #action>
            <div style="height: 50px;"></div>
            <n-button type="primary" @click="startBuildJob">开始构建</n-button>
        </template>
    </n-modal>
  </div>
</template>

<script lang="ts">
import { defineComponent, ref, onMounted, watch, computed, h } from 'vue';
import { NCard, NButton, NDataTable, NSpace, NEmpty, NModal, NSpin, NForm, NFormItem, NInput, NSelect, NSwitch, NIcon, NTooltip, NTime } from 'naive-ui';
import { HelpFilled } from "@vicons/carbon";
import { Construct as ConstructIcon, CloseCircleOutline, Eye } from "@vicons/ionicons5";
import StatusIcon from './StatusIcon.vue';
import { useStore } from 'vuex';
import { useMessage, useDialog } from 'naive-ui';
import utils from '@/utils/toolsbox';

const utools = window.utools;

export default defineComponent({
  name: 'FavoriteJobs',
  components: {
    NCard,
    NButton,
    NDataTable,
    NSpace,
    NEmpty,
    NModal,
    NSpin,
    NForm,
    NFormItem,
    NInput,
    NSelect,
    NSwitch,
    NIcon,
    NTooltip,
    NTime,
    HelpFilled,
    StatusIcon,
    CloseCircleOutline,
    ConstructIcon,
    Eye
  },
  props: {
    active: {
      type: Boolean,
      default: false
    }
  },
  setup(props) {
    const store = useStore();
    const message = useMessage();
    const dialog = useDialog();
    const favoriteJobs = ref<any[]>([]);
    const showbuildParamsModal = ref(false);
    const buildParamsLoading = ref(false);
    const currentBuildJob = ref<any>({});
    const jobParameterTypeOfInput = ref(["StringParameterDefinition", "DateParameterDefinition",
      "LabelParameterDefinition", "PersistentStringParameterDefinition","PasswordParameterDefinition", "PT_TEXTBOX"]);
    const jobParameterTypeOfTextArea = ref(["TextParameterDefinition", "PersistentTextParameterDefinition"]);
    const jobParameterTypeOfSelect = ref(["ChoiceParameterDefinition", "CascadeChoiceParameter", "BooleanParameterDefinition",
      "NodeParameterDefinition", "PersistentChoiceParameterDefinition",
      "PT_MULTI_SELECT", "PT_SINGLE_SELECT", "PT_TAG", "PT_BRANCH", "PT_BRANCH_TAG", "PT_REVISION", "PT_PULL_REQUEST"]);
    const jobParameterTypeOfCheckbox = ref(["BooleanParameterDefinition", "PersistentBooleanParameterDefinition", ""]);

    const createColumns = () => {
      return [
        {
          title: '状态',
          key: 'status',
          width: 60,
          render(row: any) {
            return h(StatusIcon, {
              name: row.icon,
              color: row.color,
              anime: row.anime
            });
          }
        },
        {
          title: '名称',
          key: 'name',
          render(row: any) {
            return h('a', {
              href: '#',
              onClick: (e: Event) => {
                e.preventDefault();
                handleJobName(row);
              },
              class: 'job-name-link'
            }, row.name);
          }
        },
        {
          title: '最后构建时间',
          key: 'lastBuildTime',
          width: 180,
          render(row: any) {
            return h(NTime, {
              time: new Date(row.lastBuildTime),
              format: 'yyyy-MM-dd HH:mm:ss'
            });
          }
        },
        {
          title: '操作',
          key: 'actions',
          width: 200,
          render(row: any) {
            return h(NSpace, null, {
              default: () => [
                h(
                  NButton,
                  {
                    strong: true,
                    secondary: true,
                    circle: true,
                    type: "success",
                    title: row.buildStatus && row.buildStatus !== 'FINISH' ? "取消" : "构建",
                    onClick: () => row.buildStatus && row.buildStatus !== 'FINISH' ? cancelBuildJob(row) : handleBuild(row),
                    style: {
                      marginLeft: "10px",
                    },
                  },
                  {
                    icon: () =>
                      h(NIcon, {
                        component: row.buildStatus && row.buildStatus !== 'FINISH' ? CloseCircleOutline : ConstructIcon,
                      }),
                  }
                ),
                h(
                  NButton,
                  {
                    strong: true,
                    secondary: true,
                    circle: true,
                    type: "success",
                    title: "取消关注",
                    onClick: () => removeFavorite(row),
                    style: {
                      marginLeft: "10px",
                    },
                  },
                  {
                    icon: () =>
                      h(NIcon, {
                        component: Eye,
                      }),
                  }
                )
              ]
            });
          }
        }
      ];
    };

    const columns = computed(() => createColumns());

    const handleBuild = async (job: any) => {
      buildParamsLoading.value = true;
      if (!job.form) {
        job.form = {};
      }
      try {
        const data = await store.dispatch("getJobAct", job.name).then(res => res.data);
        job.property = data.property;
        job.description = data.description;
        job.displayName = data.displayName;
        job.parameterProcessed = [];
        if (job.property && job.property.length > 0) {
          for (let property of job.property) {
            if (!property || !property.parameterDefinitions || property.parameterDefinitions.length === 0) {
              continue;
            }
            for (let param of property.parameterDefinitions) {
              job.form[param.name] = param.defaultParameterValue ? param.defaultParameterValue.value : '';
              await store.dispatch("handleGitParameterAct", {job: job, param: param });
              job.parameterProcessed.push(param);
            }
          }
        }
      } finally {
        buildParamsLoading.value = false;
      }

      showbuildParamsModal.value = true;
      currentBuildJob.value = job;
    };

    const startBuildJob = () => {
      store.dispatch("buildJobAct", { jobName: currentBuildJob.value.name, parameters: currentBuildJob.value.form }).then((res) => {
        const job = currentBuildJob.value;
        
        let location = res.headers.location;
        let regExp = new RegExp('^.*/item/([a-zA-Z0-9]+)/$');
        let array = regExp.exec(location);
        if (array!.length >= 2) {
          job.curBuildingQueueItemId = array![1];
        }
        if (job.curBuildingNumber) {
          delete job.curBuildingNumber;
        }
        job['buildStatus'] = 'SUBMIT';
        timingGetBuildProgress(job);
        message.success("开始构建");
        showbuildParamsModal.value = false;
      }).catch(err => {
        console.log(err);
        message.error("构建失败");
        showbuildParamsModal.value = false;
      });
    };

    const timingGetBuildProgress = async (job: any) => {
      let timing = true;
      try {
        await getBuildProgress(job);
        if (job.buildStatus === 'FINISH') {
          let notify = '构建结束！';
          timing = false;
          switch (job.buildResult) {
            case 'SUCCESS':
              notify = '构建成功！';
              break;
            case 'UNSTABLE':
              notify = '构建结束，结果不稳定！';
              break;
            case 'FAILURE':
              notify = '构建失败！';
              break;
            case 'NOT_BUILT':
              notify = '构建未开始！';
              break;
            case 'ABORTED':
              notify = '构建中止！';
              break;
          }
          utools.showNotification(job.name + notify);
        } else {
          job.progressTask = setTimeout(() => timingGetBuildProgress(job), 5000);
        }
      } catch (e) {
        timing = false;
        utools.showNotification(job.name + '构建中止！');
      } finally {
        if (!timing) {
          let data = await store.dispatch("jobDetailsAct", job.name).then(res => res.data);
          job['color'] = data.color;
          utils.jobStatusToIcon(job);
          await jobLastBuild(job);
        }
      }
    };

    const getBuildProgress = async (job: any) => {
      if (job.curBuildingNumber) {
        const params = {
          jobName: job.name,
          buildNum: job.curBuildingNumber
        };
        store.dispatch("buildHistoryAct", params).then(res => {
          let result = res.data;
          if (result.building === false && result.result) {
            job['buildStatus'] = 'FINISH';
            job['buildResult'] = result.result;
            job['anime'] = false;
          } else {
            let startTime = result.timestamp;
            let nowTime = new Date().getTime();
            let estimatedDuration = result.estimatedDuration;
            let progress = -1;
            if (estimatedDuration && estimatedDuration !== -1) {
              progress = Math.ceil(((nowTime - startTime) / estimatedDuration) * 100);
              if (progress >= 100) {
                progress = 99;
              }
            }
            job['progress'] = progress;
          }
        });
      } else {
        await store.dispatch("queueItemAct", job.curBuildingQueueItemId).then(res => {
          let result = res;
          if (result.executable && result.executable.number) {
            job['curBuildingNumber'] = result.executable.number;
            job['buildStatus'] = 'BUILDING';
            job['anime'] = true;
            job['progress'] = -1;
          }
          if (result.cancelled === true) {
            job['buildStatus'] = 'FINISH';
          }
        });
      }
    };

    const jobLastBuild = async (job: any) => {
      const params = {
        jobName: job.name,
        buildNum: "lastBuild"
      };
      try {
        const res = await store.dispatch("buildHistoryAct", params);
        const result = res.data;
        job.lastBuildTime = result.timestamp;
        if (result.changeSet && result.changeSet.items.length > 0) {
          let item = result.changeSet.items[result.changeSet.items.length - 1];
          job.lastChange = item.msg + " (" + item.authorEmail + ")";
        }
        if (result.building) {
          job.buildStatus = 'BUILDING';
          job.curBuildingNumber = result.number;
          timingGetBuildProgress(job);
        }
      } catch (error: any) {
        console.log(error, "=====", error.response.status, "jobLastBuild catch -----------------------------------------");
      }
    };

    const paramsSelectOptions = computed(() => {
      const options: any = [];
      if (currentBuildJob.value && currentBuildJob.value.parameterProcessed && currentBuildJob.value.parameterProcessed.length > 0) {
        for (const key in currentBuildJob.value.parameterProcessed) {
          const params = currentBuildJob.value.parameterProcessed[key];
          if (jobParameterTypeOfSelect.value.indexOf(params.type) !== -1) {
            for (const k1 in params.choices) {
              options.push({
                label: params.choices[k1].text,
                value: params.choices[k1].value
              });
            }
          } else {
            for (const k2 in params.choices) {
              options.push({
                label: params.choices[k2],
                value: params.choices[k2]
              });
            }
          }
        }
      }
      return options;
    });

    const loadFavorites = () => {
      console.log('Loading favorites...');
      try {
        // 使用 dbStorage 而不是 db
        let favorites = utools.dbStorage.getItem('favorite_jobs');
        console.log('Raw favorites from storage:', favorites);
        
        // 如果数据不存在，创建一个空数组并保存
        if (!favorites) {
          favorites = [];
          utools.dbStorage.setItem('favorite_jobs', JSON.stringify(favorites));
          console.log('Created new empty favorites array');
        } else {
          // 如果数据存在，需要解析 JSON 字符串
          favorites = JSON.parse(favorites);
        }
        
        favoriteJobs.value = favorites;
        console.log('Loaded favorites:', favoriteJobs.value);
      } catch (error) {
        console.error('Error loading favorites:', error);
        favoriteJobs.value = [];
      }
    };

    const removeFavorite = (job: any) => {
      // 获取现有的关注列表
      let favorites = [];
      try {
        const storedFavorites = utools.dbStorage.getItem('favorite_jobs');
        favorites = storedFavorites ? JSON.parse(storedFavorites) : [];
      } catch (error) {
        console.error('Error reading favorites:', error);
        favorites = [];
      }
      
      // 移除任务
      favorites = favorites.filter((j: any) => j.name !== job.name);
      
      // 保存到本地存储
      try {
        utools.dbStorage.setItem('favorite_jobs', JSON.stringify(favorites));
        favoriteJobs.value = favorites;
        message.success("已取消关注");
      } catch (error) {
        console.error('Error saving favorites:', error);
        message.error("操作失败");
      }
    };

    const cancelBuildJob = (job: any) => {
      dialog.info({
        title: "温馨提示",
        content: "您确定要取消，" + job.name + "任务吗？",
        positiveText: "确定",
        onPositiveClick: () => {
          if (job.curBuildingNumber) {
            store.dispatch("cancelBuildAct", { jobName: job.name, num: job.curBuildingNumber }).then(res => {
              message.success("取消任务成功");
            }).catch(err => {
              message.error("取消任务失败");
            });
          } else if (job.curBuildingQueueItemId) {
            store.dispatch("cancelQueueItemAct", job.curBuildingQueueItemId).then(res => {
              message.success("取消队列成功");
            }).catch(err => {
              message.error("取消队列失败");
            });
          }
        }
      });
    };

    const handleJobName = (job: any) => {
      const jenkinsUrl = store.getters.jenkinsUrl;
      if (!jenkinsUrl) {
        message.error("未配置 Jenkins 地址");
        return;
      }
      utools.shellOpenExternal(jenkinsUrl + '/job/' + job.name);
    };

    // 添加 watch 来监听标签页切换
    watch(() => props.active, (newVal) => {
      console.log('Tab changed, active:', newVal);
      if (newVal) {
        loadFavorites();
      }
    });

    onMounted(() => {
      console.log('Component mounted, loading favorites...');
      loadFavorites();
    });

    return {
      favoriteJobs,
      showbuildParamsModal,
      buildParamsLoading,
      currentBuildJob,
      jobParameterTypeOfInput,
      jobParameterTypeOfTextArea,
      jobParameterTypeOfSelect,
      jobParameterTypeOfCheckbox,
      paramsSelectOptions,
      columns,
      handleBuild,
      startBuildJob,
      cancelBuildJob,
      removeFavorite,
      handleJobName
    };
  }
});
</script>

<style scoped>
.favorite-jobs {
  margin-bottom: 16px;
}

.favorite-jobs-card {
  margin-bottom: 16px;
}

.job-name-link {
  font-size: 14px;
  color: #666666;
  text-decoration: none;
  cursor: pointer;
}

.job-name-link:hover {
  color: #18a058;
  text-decoration: none;
}

:deep(.n-data-table) {
  margin-top: 16px;
}

:deep(.n-data-table .n-data-table-td) {
  padding: 8px 12px;
}

:deep(.n-data-table .n-data-table-th) {
  padding: 8px 12px;
  background-color: #f5f5f5;
}

:deep(.n-data-table .n-data-table-tr) {
  transition: background-color 0.3s;
}

:deep(.n-data-table .n-data-table-tr:hover) {
  background-color: #f5f5f5;
}

:deep(.n-data-table .n-data-table-td) {
  border-bottom: 1px solid #eee;
}

:deep(.n-data-table .n-data-table-th) {
  border-bottom: 1px solid #eee;
  font-weight: 500;
}

:deep(.n-data-table .n-data-table-td:first-child),
:deep(.n-data-table .n-data-table-th:first-child) {
  padding-left: 16px;
}

:deep(.n-data-table .n-data-table-td:last-child),
:deep(.n-data-table .n-data-table-th:last-child) {
  padding-right: 16px;
}

:deep(.n-data-table .n-data-table-td) {
  font-size: 14px;
}

:deep(.n-data-table .n-data-table-th) {
  font-size: 14px;
  color: #666666;
}

:deep(.n-data-table .n-data-table-tr:last-child .n-data-table-td) {
  border-bottom: none;
}

:deep(.n-data-table .n-data-table-td .n-button) {
  padding: 4px 8px;
}

:deep(.n-data-table .n-data-table-td .n-button .n-icon) {
  font-size: 16px;
}

:deep(.n-data-table .n-data-table-td .n-space) {
  gap: 8px;
}
</style> 