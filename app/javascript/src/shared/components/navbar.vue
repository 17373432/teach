<template>
  <div class="navbar clearFloat">
    <el-menu
      :default-active="kanban"
      mode="horizontal"
      @select="handleSelect"
      background-color="#545c64"
      text-color="#fff"
      active-text-color="#ffd04b">
      <el-submenu index="1">
        <template slot="title">项目看板</template>
        <el-menu-item v-for="(project, index) in projects" :key="index" :index="'1-' + project.id">
          {{ project.name_with_namespace }}
        </el-menu-item>
      </el-submenu>

      <el-submenu index="2">
        <template slot="title">冲刺</template>
        <el-submenu
          v-for="(milestone, index) in milestones"
          :key="index"
          :index="'2-' + index">
          <template slot="title">{{ milestone.title }}</template>
          <el-menu-item
            :index="'2-' + milestone.project_id + '-' + milestone.id + '-kanban'">
            看板
          </el-menu-item>
          <el-menu-item
            :index="'2-' + milestone.project_id + '-' + milestone.id + '-burndown'">
            冲刺详情
          </el-menu-item>
        </el-submenu>
        <el-submenu index="2-new" v-if="this.projects.length">
          <template slot="title">新建冲刺</template>
          <el-menu-item v-for="(project, index) in projects" :key="index" :index="'2-new-' + project.id">
            {{ project.name }}
          </el-menu-item>
        </el-submenu>
      </el-submenu>
      <el-menu-item index="3">
        问题
      </el-menu-item>

      <el-menu-item index="4">
        班级
      </el-menu-item>

      <el-menu-item index="5">
        GitLab
        <i class="iconfont icon-link"></i>
      </el-menu-item>


      <el-submenu index="6" style="float: right">
        <template slot="title">新建问题</template>
        <el-menu-item v-for="(project, index) in projects" :key="index" :index="'6-' + project.id">
          {{ project.name_with_namespace }}
        </el-menu-item>
      </el-submenu>

    </el-menu>

    <el-dialog
      title="新建问题"
      top="50px"
      :visible.sync="dialogVisible"
      v-loading="loading"
      element-loading-text="创建中"
      width="80%">
      <new-issue ref="newIssue" :issue="newIssue"></new-issue>
      <span slot="footer" class="dialog-footer">
        <el-button @click="dialogVisible = false">取 消</el-button>
        <el-button type="primary" @click="addIssue('newIssueForm')">确 定</el-button>
      </span>
    </el-dialog>
  </div>
</template>
<script>
  import NewIssue from './new_issue.vue'
  import Issue from '../../issues/models/issue'
  import IssuesService from "../../issues/services/issues_service";
  import AlertMixin from '../../shared/components/mixins/alert'

  export default {
    mixins: [AlertMixin],
    components: {
      NewIssue,
    },
    data() {
      return {
        kanban: '1',
        issues: '2',
        gitlab: '3',
        projects: [],
        milestones: [],
        gitlabHost: '',
        dialogVisible: false,
        newIssue: new Issue(),
        // 创建 Issue 时的 loading
        loading: false
      };
    },
    mounted() {
      const navbar = document.getElementById('navbar');
      this.gitlabHost = navbar.dataset.gitlabhost;
      const projects = JSON.parse(navbar.dataset.projects);
      for (let project of projects) {
        let milestones = project.milestones;
        for (let milestone of milestones) {
          this.milestones.push(milestone);
        }
        this.projects.push({
          id: project.id,
          name: project.name,
          name_with_namespace: project.name_with_namespace,
          webUrl: project.web_url
        });
      }
    },
    updated() {
      if (this.dialogVisible) {
        const $dialog = document.getElementsByClassName('el-dialog')[0];
        const navHeight = document.getElementById('navbar').clientHeight;
        const total = document.documentElement.clientHeight;
        $dialog.style.maxHeight = (total - 100) + 'px';
        $dialog.style.overflowY = 'auto';
      }
    },
    methods: {
      handleSelect(key, keyPath) {
        if (key.startsWith('1-')) {
          let project_id = key.substr(2);
          window.location.href = `/projects/${project_id}/kanban`;
        }
        // 新建冲刺
        else if (key.startsWith('2-new')) {
          let list = key.split('-');
          let projectId = parseInt(list[list.length - 1]);
          let webUrl = this.projects.find((p) => p.id === projectId).webUrl;
          window.open(`${webUrl}/milestones/new`);
        } else if (key.startsWith('2-')) {
          let list = key.substr(2).split('-');
          if (list.length === 3) {
            let milestone_id = list[1];
            let route = list[2];
            window.location.href = `/milestones/${milestone_id}/${route}`;
          }
        } else if (key === '3') {
          window.location.href = '/issues';
        } else if (key === '4') {
          window.location.href = "/classrooms";
        } else if (key === '5') {
          window.location.href = this.gitlabHost;
        } else if (key.startsWith('6-')) {
          this.newIssue.projectId = parseInt(key.substr(2));
          this.dialogVisible = true;
        }
      },
      handleClose(done) {
        this.$confirm('确认关闭？')
          .then(_ => {
            done();
          })
          .catch(_ => {
          });
      },
      addIssue(formName) {
        this.$refs['newIssue'].$refs[formName].validate((valid) => {
          if (valid) {
            this.newIssue.state = 'opened';
            if (window.eventhub) {
              // issues 和 boards 中创建
              this.dialogVisible = false;
              window.eventhub.$emit('addNewIssue', this.newIssue);
            }
            else {
              // 其他页面通过导航栏创建
              this.addNewIssue(this.newIssue);
            }
            this.newIssue = new Issue();
          } else {
            return false;
          }
        })
      },
      addNewIssue(issue) {
        this.loading = true;
        const navbar = document.getElementById('navbar');
        const issuesService = new IssuesService({
          issuesEndpoint: navbar.dataset.issuesEndpoint
        });
        issuesService
          .newIssue(issue.toObj())
          .then(res => res.data)
          .then((data) => {
            let issue = Issue.valueOf(data);
            this.success('创建成功');
            this.loading = false;
            this.dialogVisible = false;
          })
          .catch(e => {
            this.alert('创建失败');
            this.loading = false;
            this.dialogVisible = false;
          })
      }
    }
  }
</script>
<style scoped>
  .navbar a {
    text-decoration: none;
  }

  .navbar i {
    font-size: 14px;
  }
</style>