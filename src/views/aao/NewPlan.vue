<template>
  <div>
    <content-title :title="titleInfo.title" :subtitle="titleInfo.subtitle"/>
    <div class="content-style">
      <a-form :form="form" @submit="handleSubmit">
        <a-form-item label="学年：" :label-col="{ span: 5 }" :wrapper-col="{ span: 12 }"
                     :validate-status="year.validateStatus"
                     :help="year.errorMsg">
          <a-input-number size="small" :min="2000" :max="2999" :defaultValue="2019" :value="year.startValue"
                          @change="handleStartNumberChange"/>
          -
          <a-input-number size="small" :min="2000" :max="2999" :defaultValue="2020" :value="year.endValue"
                          @change="handleEndNumberChange"/>
        </a-form-item>
        <a-form-item label="学期：" :label-col="{ span: 5 }" :wrapper-col="{ span: 12 }">
          <a-radio-group :defaultValue="0" v-model="submitData.term">
            <a-radio :value="0">第一学期</a-radio>
            <a-radio :value="1">第二学期</a-radio>
          </a-radio-group>
        </a-form-item>
        <a-form-item label="开课院系部：" :label-col="{ span: 5 }" :wrapper-col="{ span: 12 }">
          <a-select v-decorator="['teaching_department',{rules: [{ required: true, message: '请选择授课部门！' }]}]"
                    placeholder="选择开课院系部" :loading="teaching_department_loading">
            <a-select-option v-for="teaching_department in formData.teaching_department" :key="teaching_department.id"
                             :value="teaching_department.id">
              {{teaching_department.name}}
            </a-select-option>
          </a-select>
        </a-form-item>
        <a-form-item label="教育层次：" :label-col="{ span: 5 }" :wrapper-col="{ span: 12 }">
          <a-select v-decorator="['educational_level',{rules: [{ required: true, message: '请选择教育层次！' }]}]"
                    placeholder="选择教育层次" :loading="educational_levels_loading">
            <a-select-option v-for="educational_levels in formData.educational_levels" :key="educational_levels.id"
                             :value="educational_levels.id">
              {{educational_levels.educationalLevel}}
            </a-select-option>
          </a-select>
        </a-form-item>
        <a-form-item label="征订计划：" :label-col="{ span: 5 }" :wrapper-col="{ span: 12 }">
          <a-upload name="planFile" :multiple="false" accept=".xls,.xlsx" :beforeUpload="beforeUpload"
                    :customRequest="customRequest"
                    v-decorator="['plan',{rules: [{ required: true, message: '请上传征订计划！' }]}]">
            <a-button>
              <a-icon type="upload"/>
              选择文件（支持扩展名：.xls .xlsx）
            </a-button>
          </a-upload>
          <a @click="downTemplate">下载征订计划模板</a>
        </a-form-item>
        <a-form-item :wrapper-col="{ span: 12, offset: 5 }">
          <a-button type="primary" html-type="submit" :disabled="disabledSubmitBtn" :loading="loading">提交</a-button>
        </a-form-item>
      </a-form>
    </div>
  </div>
</template>

<script>
  import ContentTitle from "@/components/ContentTitle";
  import {Download, Get, Post} from "../../axios";
  import Api from '../../api'

  export default {
    name: "NewPlan",
    components: {ContentTitle},
    data() {
      return {
        form: this.$form.createForm(this),
        titleInfo: {
          title: '新增计划',
          subtitle: '新增一个教材征订计划',
        },
        //学年
        year: {
          //开始学年
          startValue: 2019,
          //结束学年
          endValue: 2020
        },
        //正在提交
        loading: false,
        //正在加载授课部门
        teaching_department_loading: true,
        //正在加载教育层次
        educational_levels_loading: true,
        //禁用提交按钮
        disabledSubmitBtn: false,
        //表单数据
        formData: {
          //授课部门下拉数据
          teaching_department: [],
          //教育层次下拉数据
          educational_levels: []
        },
        //表单提交数据
        submitData: {
          //学期 0 1
          term: 0,
          year: '2019-2020',
          fileId: null
        }
      }
    },
    methods: {
      /**
       * 处理表单提交
       * @param e event
       */
      handleSubmit(e) {
        e.preventDefault();
        this.form.validateFields((err, values) => {
          if (!err) {
            //组装提交数据
            this.submitData.teachingDepartment = values.teaching_department;
            this.submitData.educationalLevel = values.educational_level;
            this.loading = true;
            this.disabledSubmitBtn = true;
            Post(Api.postPlan)
              .withSuccessCode(201)
              .withURLSearchParams(this.submitData)
              .do(response => {
                this.$message.success(response.data.msg);
                this.$router.push("/plan_list");
              })
              .doAfter(() => {
                this.loading = false;
                this.disabledSubmitBtn = false;
              })
          }
        });
      },
      /**
       * 处理起始学年更改
       * @param value 新值
       */
      handleStartNumberChange(value) {
        //起始年份是否小于结束年份
        this.year = {
          startValue: value,
          endValue: value + 1
        };
        this.submitData.year = this.year.startValue + '-' + this.year.endValue;

      },
      /**
       * 处理结束学年更改
       * @param value 新值
       */
      handleEndNumberChange(value) {
        this.year = {
          endValue: value,
          startValue: value - 1
        };
        this.submitData.year = this.year.startValue + '-' + this.year.endValue;
      },
      /**
       * 检查文件是否是Excel
       * @param file 文件
       * @param fileList 文件列表
       * @returns {boolean} 是否允许上传文件
       */
      beforeUpload(file, fileList) {
        let exName = file.name.slice((file.name.lastIndexOf(".") - 1 >>> 0) + 2);
        if (exName === 'xls' || exName === 'xlsx') {
          this.disabledSubmitBtn = false;
          return true;
        } else {
          this.disabledSubmitBtn = true;
          this.$notification['error']({
            message: '文件不正确，请重新选择',
            description: `文件${file.name}不是正确的Excel文件，支持的扩展名：xls或xlsx`,
          });
          return false;
        }
      },
      /**
       * 初始化表单数据
       */
      initFormData() {
        Get(Api.getTeachingDepartments).do(response => {
          this.formData.teaching_department = response.data.data;
        }).doAfter(() => {
          this.teaching_department_loading = false;
        });
        Get(Api.getEducationalLevels).do(response => {
          this.formData.educational_levels = response.data.data;
        }).doAfter(() => {
          this.educational_levels_loading = false;
        });
      },
      /**
       * 下载执行计划模板
       */
      downTemplate() {
        Download(Api.getDownloadPlanTemplate, headers => {
          return "征订计划模板.xlsx"
        });
      },
      /**
       * 文件上传
       * @param data data
       */
      customRequest(data) {
        let progress = {percent: 0};
        Post(Api.postPlanFile)
          .withSuccessCode(201)
          .withErrorStartMsg("上传失败：")
          .withFormData({planFile: data.file}, true, p => {
            progress.percent = p;
            if (progress.percent < 100) {
              data.onProgress(progress)
            } else {
              data.onSuccess();
            }
          })
          .do(response => {
            this.$message.success('上传成功');
            this.submitData.fileId = response.data.data;
          })
      }
    },
    created() {
      //初始化表单数据
      this.initFormData();
    }
  }
</script>

<style scoped>

</style>
