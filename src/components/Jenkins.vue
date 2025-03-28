<script lang="ts">
import { NButton, NIcon,NTime,SelectOption, useMessage, useDialog } from "naive-ui";
import { Construct as ConstructIcon, CloseCircleOutline, EyeOutline, Eye } from "@vicons/ionicons5";
import { ChangeCatalog } from "@vicons/carbon";
import { h, defineComponent, ref, computed, nextTick, watch, onMounted } from "vue";
import { useStore } from 'vuex'
import utils from "@/utils/toolsbox";
import StatusIcon from "@/components/StatusIcon.vue"
import FavoriteJobs from './FavoriteJobs.vue'

const utools = window.utools

type TableFunArg = (row: any, index: number) => void
type TableFunArg2 = (row: any) => void

const isJobFavorited = (jobName: string) => {
  try {
    const favorites = utools.dbStorage.getItem('favorite_jobs');
    if (!favorites) return false;
    const favoriteJobs = JSON.parse(favorites);
    return favoriteJobs.some((job: any) => job.name === jobName);
  } catch (error) {
    console.error('Error checking favorite status:', error);
    return false;
  }
};

const createColumns = (
  { buildJobParamsDialog }: { buildJobParamsDialog: TableFunArg },
  { cancelBuildJob }: { cancelBuildJob: TableFunArg2 },
  { viewLog }: { viewLog: TableFunArg2 },
  { handleJobName }: { handleJobName: TableFunArg2 },
  { addToFavorites }: { addToFavorites: TableFunArg2 }
) => {
  return [
    {
      title: "状态",
      width: 48,
      render(row: any) {
          return h(StatusIcon, {
              name: row.icon,
              color: row.color,
              anime: row.anime
          })
      }
    },
    {
      title: "名称",
      key: "name",
      width: 180,
      fixed: "left",
      render(row: any) {
          return h(
              NButton,
              {
                  text: true,
                  tag: 'a',
                  target: '_blank',
                  onClick: () =>handleJobName(row),
              },
              { default: () => row.name }
          )
      },
    },
    {
      title: "构建时间",
      key: "lastBuildTime",
      width: 180,
      render (row: any) {
        if(!(row.lastBuildTime) || row.lastBuildTime === "N/A") {
            return h('span', ["N/A"])
        } else {
            return h(NTime, {
                time: row.lastBuildTime,
                format: 'yyyy-MM-dd HH:mm:ss'
            })
        }
      }
    },
    {
      title: "操作",
      key: "action",
      width: 200,
      render(row: any, index: number) {
          const ops = [];
          if(row.buildStatus && row.buildStatus !== 'FINISH') {
              ops[0] = h(
                    NButton,
                    {
                    strong: true,
                    secondary: true,
                    circle: true,
                    type: "success",
                    title: "取消",
                    onClick: () => cancelBuildJob(row),
                    },
                    {
                    icon: () =>
                        h(NIcon, {
                        component: CloseCircleOutline,
                        }),
                    }
                )
          } else {
            ops[0] = h(
                NButton,
                {
                strong: true,
                secondary: true,
                circle: true,
                type: "success",
                title: "构建",
                onClick: () => buildJobParamsDialog(row, index),
                },
                {
                icon: () =>
                    h(NIcon, {
                    component: ConstructIcon,
                    }),
                }
            )
          }
          ops[1] = h(
                NButton,
                {
                strong: true,
                secondary: true,
                circle: true,
                type: "success",
                title: "查看日志",
                style: {
                    marginLeft: "10px",
                },
                onClick: () => viewLog(row),
                },
                {
                icon: () =>
                    h(NIcon, {
                    component: ChangeCatalog,
                    }),
                }
            )
          ops[2] = h(
                NButton,
                {
                strong: true,
                secondary: true,
                circle: true,
                type: "success",
                title: isJobFavorited(row.name) ? "取消关注" : "关注",
                style: {
                    marginLeft: "10px",
                },
                onClick: () => addToFavorites(row),
                },
                {
                icon: () =>
                    h(NIcon, {
                    component: isJobFavorited(row.name) ? Eye : EyeOutline,
                    }),
                }
            )
          return ops;
      },
    },
  ];
};

const jenkinsList = () => {
    const list = utils.getConfigList()
    let jks:any[] = []
    list.forEach((item) => {
        jks.push({
            label: item.data.name,
            value: item._id,
            data: item.data
        })
    })
    return jks
}

export default defineComponent({
  name: "Jenkins",
  components: { StatusIcon, EyeOutline, Eye, FavoriteJobs },
  setup() {
    const store = useStore()
    const message = useMessage()
    const dialog = useDialog()
    const showbuildParamsModal = ref(false)
    const buildParamsLoading = ref(false)
    const showLogModal = ref(false)
    const currentBuildJob = ref<any>({})
    const jobParameterTypeOfSelectNoChoice = ref(["PT_MULTI_SELECT", "PT_SINGLE_SELECT", "PT_TAG", "PT_BRANCH", "PT_BRANCH_TAG", "PT_REVISION", "PT_PULL_REQUEST"])
    const buildLog = ref("")
    const logJobName = ref("")
    const logLoading = ref(false)
    const configList = ref<any>(jenkinsList());
    const jkName = ref(configList?.value[0]?.value || null)
    const jobs = ref<any>([{}])
    const jobName = ref("")
    const viewNames = ref<any>([{}])
    const viewName = ref(store.state.config?.defaultView || 'all')
    
    store.dispatch("configAct",configList?.value[0]?.data)

    const buildJobParamsDialog = async (job: any, index: number) => {
        buildParamsLoading.value = true
        if (!job.form) {
          job.form = {}
        }
        try {
          // const job = {}
          const data = await store.dispatch("getJobAct",job.name).then(res => res.data);
          job.property = data.property
          job.description = data.description
          job.displayName = data.displayName
          job.parameterProcessed = []
          if (job.property && job.property.length > 0) {
          for (let property of job.property) {
            if (!property || !property.parameterDefinitions || property.parameterDefinitions.length === 0) {
              continue
            }
            for (let param of property.parameterDefinitions) {
              job.form[param.name] = param.defaultParameterValue ? param.defaultParameterValue.value : '';
              await store.dispatch("handleGitParameterAct", {job: job, param: param });
              job.parameterProcessed.push(param)
            }
          }
        }

        } finally {
          buildParamsLoading.value = false
        }

        showbuildParamsModal.value = true
        currentBuildJob.value = job
    };

    const startBuildJob = () => {
      store.dispatch("buildJobAct", { jobName: currentBuildJob.value.name,parameters: currentBuildJob.value.form }).then((res) => {
          const job = currentBuildJob.value
          
          let location = res.headers.location
          let regExp = new RegExp('^.*/item/([a-zA-Z0-9]+)/$')
          let array = regExp.exec(location);
          if (array!.length >= 2) {
              job.curBuildingQueueItemId = array![1]
          }
          if (job.curBuildingNumber) {
              delete job.curBuildingNumber
          }
          job['buildStatus'] = 'SUBMIT'
          timingGetBuildProgress(job)
          message.success("开始构建")
          showbuildParamsModal.value = false
      }).catch(err => {
          console.log(err);
          message.error("构建失败")
          showbuildParamsModal.value = false
      })
    }

    const cancelBuildJob = (job: any) => {
        dialog.info({
            title: "温馨提示",
            content: "您确定要取消，" + job.name + "任务吗？",
            positiveText: "确定",
            onPositiveClick: () => {
               if (job.curBuildingNumber) {
                   store.dispatch("cancelBuildAct", { jobName: job.name, num: job.curBuildingNumber}).then(res => {
                        message.success("取消任务成功")
                   }).catch(err => {
                        message.error("取消任务失败")
                   })
                } else if (job.curBuildingQueueItemId) {
                    store.dispatch("cancelQueueItemAct", job.curBuildingQueueItemId).then(res => {
                        message.success("取消队列成功")
                    }).catch(err => {
                        message.error("取消队列失败")
                    })
                }
                
            }
        })
    }

    const viewLog = (job: any) => {
        buildLog.value = ""
        // this.currentJob = job
        logJobName.value = job.name
        job.fetchedSize = 0
        showLogModal.value = true
        logLoading.value = true
        if (job.buildStatus && (job.buildStatus === 'SUBMIT' || job.buildStatus === 'BUILDING')) {
            timingGetBuildConsole(job)
            logLoading.value = false
        } else {
            store.dispatch("buildConsoleAct", { jobName: job.name, number: 'lastBuild', start: 0 }).then(res => {
                buildLog.value = res.data
                logLoading.value = false
            }).catch(err => {
                console.log(err, "viewLog-------------------")
                if(err.response.status == 404) {
                    buildLog.value = "暂无日志"
                }
                logLoading.value = false
            })
        }
    };

    const timingGetBuildConsole = async (job: any) => {
      if (job.buildStatus === 'FINISH') {
        return false
      }
      if (!job.curBuildingNumber) {
        job.buildConsoleTask = setTimeout(() => timingGetBuildConsole(job), 5000)
        return true
      }
      try {
        return await store.dispatch("buildConsoleAct", { jobName: job.name, number: 'lastBuild', start: 0 }).then(res => {
          if (res.headers['content-length'] === 0) {
            job.buildConsoleTask = setTimeout(() => timingGetBuildConsole(job), 5000)
            return true
          }
          buildLog.value = buildLog.value + res.data
          let moreData = res.headers['x-more-data']
          job.fetchedSize = res.headers['x-text-size']
          if (moreData && moreData !== false) {
            job.buildConsoleTask = setTimeout(() => timingGetBuildConsole(job), 3000)
          }
        })
      } catch (e) {
        return false
      }
    }

    const setProgress = (job:any) => {
    //   let row = document.querySelectorAll('.el-table__fixed .el-table__row > td:nth-child(2)').item(job.index);
      let row = document.querySelectorAll('.n-data-table-table .n-data-table-tbody .n-data-table-tr > td:nth-child(2)').item(job.index);
      if (row) {
        if (job.buildStatus === 'FINISH') {
          row.removeAttribute('style');
        } else {
          row.setAttribute('style', 'background: linear-gradient(to right, #D9ECFF ' + job.progress + '%,#ffffff 0%)');
        }
      } else {
        console.log('xxxxxxxxxx, row not exist')
      }
    }

    /**
     * 获取构建进度
     * @param job
     */
    const getBuildProgress = async (job: any) => {
      if (job.curBuildingNumber) {
          const params = {
            jobName:job.name,
            buildNum:job.curBuildingNumber
          }
          store.dispatch("buildHistoryAct",params).then(res => {
          let result = res.data
          if (result.building === false && result.result) {
            job['buildStatus'] = 'FINISH'
            job['buildResult'] =result.result
            job['anime'] = false
            setProgress(job)
          } else {
            let startTime = result.timestamp
            let nowTime = new Date().getTime()
            let estimatedDuration = result.estimatedDuration
            let progress = -1
            if (estimatedDuration && estimatedDuration !== -1) {
              progress = Math.ceil(((nowTime - startTime) / estimatedDuration) * 100)
              if (progress >= 100) {
                progress = 99
              }
            }
            job['progress'] = progress
            setProgress(job)
          }
        })
      } else {
        await store.dispatch("queueItemAct",job.curBuildingQueueItemId).then(res => {
          let result = res
          if (result.executable && result.executable.number) {
            job['curBuildingNumber'] = result.executable.number
            job['buildStatus'] = 'BUILDING'
            job['anime'] = true
            job['progress'] = -1
          }
          if (result.cancelled === true) {
            job['buildStatus'] = 'FINISH'
          }
        })
      }
    }

    /**
     * 定时获取构建进度
     */
    const timingGetBuildProgress = async (job: any) => {
      let timing = true
      try {
        await getBuildProgress(job)
        if (job.buildStatus === 'FINISH') {
          let notify = '构建结束！'
          timing = false
          switch (job.buildResult) {
            case 'SUCCESS':
              notify = '构建成功！'
              break
            case 'UNSTABLE':
              notify = '构建结束，结果不稳定！'
              break
            case 'FAILURE':
              notify = '构建失败！'
              break
            case 'NOT_BUILT':
              notify = '构建未开始！'
              break
            case 'ABORTED':
              notify = '构建中止！'
              break
          }
          utools.showNotification(job.name + notify)
        } else {
          job.progressTask = setTimeout(() => timingGetBuildProgress(job), 5000)
        }
      } catch (e) {
        timing = false
        utools.showNotification(job.name + '构建中止！')
      } finally {
        if (!timing) {
          let data = await store.dispatch("jobDetailsAct", job.name).then(res => res.data)// this.jenkins.getJob(job.name).then(res => res.data);
          job['color'] = data.color
          utils.jobStatusToIcon(job)
          await jobLastBuild(job)
        }
      }
    }

    /**
     * 获取任务上一次构建内容
     * @param job
     * @returns {Promise<void>}
     */
    const jobLastBuild = async (job: any) => {
        const params = {
            jobName:job.name,
            buildNum:"lastBuild"
        }
        try {
            const res = await store.dispatch("buildHistoryAct",params)
            const result = res.data
            job.lastBuildTime = result.timestamp;
            if (result.changeSet && result.changeSet.items.length > 0) {
                let item = result.changeSet.items[result.changeSet.items.length - 1];
                job.lastChange = item.msg + " (" + item.authorEmail + ")";
            }
            if (result.building) {
                job.buildStatus = 'BUILDING'
                job.curBuildingNumber = result.number
                timingGetBuildProgress(job)
            }
        } catch (error: any) {
            console.log(error,"=====",error.response.status,"jobLastBuild catch -----------------------------------------")
        }
    }

    const jobsList = (viewName: string) => {
      console.log('jobList,start', viewName)
      let start = new Date().getTime()
        store.dispatch("jobsAct",viewName).then(async res => {
            console.log('jobsAct cost', new Date().getTime()-start)
            const jobList = res.jobs
          start = new Date().getTime()
            for (let job of jobList) {
                utils.jobStatusToIcon(job)
            }
          console.log('finish jobLastBuild info', new Date().getTime()-start)
            jobs.value = jobList
            store.dispatch("listLoadingAct", false)

        }).catch(err => {
          store.dispatch("listLoadingAct", false)
          message.warning("加载列表失败！")
        })
    }

    const viewNameList = () => {
        store.dispatch("baseInfoAct").then(res => {
            const vns = res.views;
            const selectVns = []
            for (let vn of vns) {
                selectVns.push({
                    label: vn.name,
                    value: vn.name,
                })
            }
            viewNames.value = selectVns
        })
    }

    const handleJKNameValue = (value: string, option: SelectOption) => {
        store.dispatch("configAct",option.data)
        store.dispatch("listLoadingAct", true)
        viewNameList()
        // 使用配置的默认视图
        const defaultView = (option.data as any)?.defaultView || 'all'
        viewName.value = defaultView
        jobsList(defaultView)
    }

    const handleViewNameValue = (value: string, option: SelectOption) => {
        jobsList(value)
    }

    const refreshJobsList = () => {
        jobsList(viewName.value)
        setTimeout(() => {
            message.success("刷新成功")
        }, 100)
    }

    const handleJobName = (job: any) => {
        utools.shellOpenExternal(store.getters.jenkinsUrl + '/job/' + job.name)
    }

    const handleCloseLogModel = () => {
        showLogModal.value = false
    }

    watch(buildLog, () => {
        nextTick(() => {
          const logView = document.getElementById('logView')
          logView!.scrollTop = logView!.scrollHeight
        })
    })

    const filterJobList = computed(()=> {
        let filter = jobs.value.filter((e:any) => e?.name?.match(jobName.value));
        for (let i = 0; i < filter.length; i++) {
            filter[i].index = i
        }
        return filter;
    })

    const showLoading = computed(() => {
      return store.getters.listLoading
    })

    const paramsSelectOptions = computed(() => {
      const options:any = []
      if(currentBuildJob.value && currentBuildJob.value.parameterProcessed && currentBuildJob.value.parameterProcessed.length > 0) {
        for (const key in currentBuildJob.value.parameterProcessed) {
          const params = currentBuildJob.value.parameterProcessed[key]
          if(jobParameterTypeOfSelectNoChoice.value.indexOf(params.type) !== -1) {
            for(const k1 in params.choices){
              options.push({
                label: params.choices[k1].text,
                value: params.choices[k1].value
              })
            }
          } else {
            for(const k2 in params.choices){
              options.push({
                label: params.choices[k2],
                value: params.choices[k2]
              })
            }
          }
        }
      }
      return options
    })

    const addToFavorites = (job: any) => {
      console.log('Adding to favorites:', job);
      
      // 获取现有的关注列表
      let favorites = [];
      try {
        const storedFavorites = utools.dbStorage.getItem('favorite_jobs');
        favorites = storedFavorites ? JSON.parse(storedFavorites) : [];
        console.log('Current favorites:', favorites);
      } catch (error) {
        console.error('Error reading favorites:', error);
        favorites = [];
      }
      
      // 检查是否已经关注
      const existingIndex = favorites.findIndex((j: { name: string }) => j.name === job.name);
      if (existingIndex !== -1) {
        // 如果已关注，则取消关注
        favorites.splice(existingIndex, 1);
        message.success("已取消关注");
      } else {
        // 如果未关注，则添加关注
        const favoriteJob = {
          name: job.name,
          icon: job.icon || 'help',
          color: job.color || '#666666',
          anime: job.anime || false,
          lastBuildTime: job.lastBuildTime || null,
          buildStatus: job.buildStatus || null,
          curBuildingNumber: job.curBuildingNumber || null,
          curBuildingQueueItemId: job.curBuildingQueueItemId || null
        };
        favorites.push(favoriteJob);
        message.success("已添加到关注列表");
      }
      
      // 保存到本地存储
      try {
        utools.dbStorage.setItem('favorite_jobs', JSON.stringify(favorites));
        console.log('Successfully saved favorites:', favorites);
        // 强制更新视图
        nextTick(() => {
          // 重新加载任务列表以更新图标
          loadJobs();
        });
      } catch (error) {
        console.error('Error saving favorites:', error);
        message.error("操作失败");
      }
    };

    const loadJobs = async () => {
      try {
        store.dispatch("listLoadingAct", true);
        const response = await store.dispatch("jobsAct", viewName.value);
        const jobList = response.jobs;
        for (let job of jobList) {
          utils.jobStatusToIcon(job);
        }
        jobs.value = jobList;
      } catch (error) {
        console.error("加载任务列表失败", error);
      } finally {
        store.dispatch("listLoadingAct", false);
      }
    };

    const loadViews = async () => {
      try {
        const response = await store.dispatch("baseInfoAct");
        const vns = response.views;
        const selectVns = [];
        for (let vn of vns) {
          selectVns.push({
            label: vn.name,
            value: vn.name,
          });
        }
        viewNames.value = selectVns;
      } catch (error) {
        console.error("加载视图列表失败", error);
      }
    };

    onMounted(async () => {
      await loadViews();
      // 只在切换到任务标签页时加载任务列表
      if (viewName.value === "all") {
        // 使用当前配置的默认视图
        const currentConfig = store.state.config;
        const defaultView = currentConfig?.defaultView || 'all';
        viewName.value = defaultView;
        await loadJobs();
      }
    });

    watch(viewName, async (newVal) => {
      if (newVal) {
        await loadJobs();
      }
    });

    return {
      jobParameterTypeOfSeparator: ref(["ParameterSeparatorDefinition"]),
      jobParameterTypeOfFile: ref(["FileParameterDefinition", "PatchParameterDefinition"]),
      jobParameterTypeOfTextArea: ref(["TextParameterDefinition", "PersistentTextParameterDefinition"]),
      jobParameterTypeOfCheckbox: ref(["BooleanParameterDefinition", "PersistentBooleanParameterDefinition", ""]),
      jobParameterTypeOfInput: ref(["StringParameterDefinition", "DateParameterDefinition",
        "LabelParameterDefinition", "PersistentStringParameterDefinition","PasswordParameterDefinition", "PT_TEXTBOX"]),
      jobParameterTypeOfSelect: ref(["ChoiceParameterDefinition", "CascadeChoiceParameter", "BooleanParameterDefinition",
        "NodeParameterDefinition", "PersistentChoiceParameterDefinition",
        "PT_MULTI_SELECT", "PT_SINGLE_SELECT", "PT_TAG", "PT_BRANCH", "PT_BRANCH_TAG", "PT_REVISION", "PT_PULL_REQUEST"]),
      jobParameterTypeOfSelectNoChoice,
      configList,
      jkName,
      viewName,
      jobName,
      viewNames,
      filterJobList,
      columns: createColumns( { buildJobParamsDialog }, { cancelBuildJob }, { viewLog }, { handleJobName }, { addToFavorites } ),
      handleJKNameValue,
      handleViewNameValue,
      refreshJobsList,
      handleCloseLogModel,
      handleJobName,
      buildLog,
      jobsList,
      viewNameList,
      showbuildParamsModal,
      buildParamsLoading,
      currentBuildJob,
      startBuildJob,
      paramsSelectOptions,
      showLogModal,
      logJobName,
      logLoading,
      showLoading,
      addToFavorites,
      loadJobs,
      loadViews
    };
  },
  beforeMount() {
    console.log('before jenkins mount', !this.$store.getters.listLoading)
    if(!this.$store.getters.listLoading) {
      this.$store.dispatch("listLoadingAct", true)
      this.viewNameList()
      // 使用配置的默认视图
      const currentConfig = this.$store.state.config;
      const defaultView = currentConfig?.defaultView || 'all';
      this.viewName = defaultView;
      this.jobsList(defaultView);
    }
  },
  mounted() {
    console.log("mounted----------------------");
  }
});
</script>

<template>
      <n-grid x-gap="12" :cols="4">
        <n-gi>
          <n-select
            v-model:value="jkName"
            :options="configList" 
            @update:value="handleJKNameValue"/>
        </n-gi>
        <n-gi>
          <n-select 
            v-model:value="viewName" 
            :options="viewNames" 
            @update:value="handleViewNameValue"/>
        </n-gi>
        <n-gi>
          <n-input v-model:value="jobName" type="text" placeholder="任务名称" clearable />
        </n-gi>
        <n-gi>
          <n-button type="success" @click="refreshJobsList">刷新列表</n-button>
        </n-gi>
      </n-grid>
      <n-data-table
        :loading="showLoading"
        :columns="columns"
        :data="filterJobList"
        class="data-table"/>
      
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
    <!-- 日志弹出框 -->
    <n-modal v-model:show="showLogModal" 
        preset="dialog" 
        title="Dialog" 
        @close="handleCloseLogModel"
        :close-on-esc="false"
        :mask-closable="false"
        :showIcon="false"
        style="width: 750px;height:450px;">
        <template #header>
            <div>查看日志({{logJobName}})</div>
        </template>
        <n-space vertical>
          <n-spin :show="logLoading">
            <pre id="logView" class="console-output" v-html="buildLog"></pre>
            <template #description>
              加载日志。。。
            </template>
          </n-spin>
        </n-space>
        
    </n-modal>
</template>

<style scoped>
.data-table {
  margin-top: 10px;
}
pre, code, kbd, samp, tt {
    font-size: var(--font-size-monospace);
}
pre {
    white-space: pre-wrap;
    word-wrap: break-word;
    margin: 0 0 var(--section-padding);
    padding: 0.8rem 1rem;
    border-radius: 10px;
    background-color: var(--pre-background);
    color: var(--pre-color) !important;
    font-family: var(--font-family-mono) !important;
    font-weight: 500 !important;
    line-height: 1.66 !important;
}
.console-output {
    max-height: 360px;
    overflow-y: auto;
    background-color: #f2f2f2;
}
</style>