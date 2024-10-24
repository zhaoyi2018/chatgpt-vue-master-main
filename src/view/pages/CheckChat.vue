<template>
    <el-container class="background-container">
        <!-- 页面头部 -->
        <el-header class="header-container" style="height: 60px;">
            <div class="header-left">
                <el-button type="text" icon="el-icon-back" @click="goToHome">返回主页</el-button>
            </div>
            <div class="header-right">
                <img src="../../imgs/logo1.png" class="logo" />
            </div>
        </el-header>
        
        <el-main style="padding: 0px;">
            <el-container class="main-container">
                <!-- 左侧边栏 -->
                <el-aside width="200px" class="aside-container">
                    <el-header class="aside-header">
                        <!-- 新建条款按钮 -->
                        <el-button type="primary" icon="el-icon-plus" :disabled="!hasCompareDone" @click="newChat" round>新条款</el-button>
                    </el-header>
                    
                    <!-- 历史记录列表 -->
                    <el-menu @select="handleSelect" class="custom-scrollbar"
                        style="background-color: #f2fbff; justify-content: center;  height:70vh; margin-bottom:20px; overflow-x: hidden; ">
                        <el-menu-item v-for="(question, index) in historyArrlist" :key="index" :index="index.toString()"
                            class="menu-item-history" @click="historyChat(question, index)">
                            <div slot="title" @mouseover="showDeleteButton(index)"
                                @mouseleave="hideDeleteButton(index)" class="menu-item-wrapper">
                                <!-- {{ getUserContent(question.history) }} -->
                                <div style="width: 120px; white-space: nowrap; overflow: hidden; text-overflow: ellipsis;">{{ question.name }}</div>
                                <el-button v-show="question.showDeleteButton" type="text" icon="el-icon-close"
                                    @click.stop="deleteItem(index)"
                                    style="position: absolute; right: 5px; top: 55%; transform: translateY(-51%);"></el-button>
                            </div>
                        </el-menu-item>
                    </el-menu>
                </el-aside>
                
                <!-- 右侧主要内容区 -->
                <el-container>
                    <el-main>
                        <!-- 逻辑1: 新建条款下的步骤显示 -->
                        <template v-if="activeStep !== 4">
                            <el-main style="display: flex; flex-direction: column; justify-content: center; align-items: center; height: 90%;">
                                <!-- 模块1: 步骤条 -->
                                <el-steps style="width: 800px; margin-bottom: 40px;" :active="activeStep" finish-status="success">
                                    <el-step title="选择标准条款" />
                                    <el-step title="上传待审核条款" />
                                    <el-step title="新文档分析中" />
                                    <el-step title="比对完成" />
                                </el-steps>
                                
                                <!-- 模块2: 根据步骤显示不同内容 -->
                                <div style="width: 800px;">
                                    <!-- 步骤1: 选择标准条款 -->
                                    <div v-if="activeStep === 1" style="display: flex; align-items: center; justify-content: space-between;">
                                        <el-select placeholder="请选择标准条款" v-model="value" style="width: calc(100% - 110px); margin-right: 10px;">
                                            <el-option v-for="item in options" :key="item.value" :label="item.label" :value="item.value">
                                            </el-option>
                                        </el-select>
                                        <!-- 下一步按钮 -->
                                        <el-button type="primary" icon="el-icon-arrow-right" @click="activeStep += 1" style="width: 100px;">下一步</el-button>
                                    </div>
                                    
                                    <!-- 步骤2: 上传待审核条款 -->
                                    <div v-else-if="activeStep === 2" style="display: flex; justify-content: center; align-items: center; height: 100%;">
                                        <el-upload class="upload-demo" action :http-request="uploadFile" :disabled="value == ''" ref="upload"
                                            :show-file-list="false" style="width: 300px;"> <!-- 设置一个固定宽度 -->
                                            <el-button type="primary" icon="el-icon-upload" style="width: 100%;">上传新条款</el-button>
                                        </el-upload>
                                    </div>
                                    
                                    <!-- 步骤3: 新文档分析中 -->
                                    <div v-else-if="activeStep === 3">
                                        <el-progress :percentage="10" :color="customColors"></el-progress>
                                        <p>新文档分析中，请稍候...</p>
                                    </div>
                                </div>
                            </el-main>
                        </template>
                        
                        <!-- 逻辑2: 比对结果表格 -->
                        <template v-else>
                            <!-- 比对结果表格 -->
                            <el-main style="padding: 28px 0px 0px 20px;">
                                <el-table 
                                    ref="clauseTable"
                                    :data="clauseTableData" 
                                    style="width: 100%;"
                                    :height="'70vh'"
                                    show-overflow-tooltip>
                                    <!-- 表格列定义 -->
                                    <el-table-column type="index" label="序号" min-width="5%" align="center">
                                    </el-table-column>
                                    <el-table-column prop="institution" label="制度要素" min-width="10%" show-overflow-tooltip>
                                    </el-table-column>
                                    <el-table-column prop="clause_number" label="法规条款" min-width="15%" show-overflow-tooltip>
                                    </el-table-column>
                                    <el-table-column prop="original_text" label="法规原文" min-width="20%" show-overflow-tooltip>
                                    </el-table-column>
                                    <el-table-column prop="related_chunks.content" label="企业制度" min-width="25%" show-overflow-tooltip>
                                    </el-table-column>
                                    <el-table-column prop="review_result" label="检查结果" min-width="5%" show-overflow-tooltip>
                                    </el-table-column>
                                    <el-table-column prop="advice" label="修改建议" min-width="20%" align="center" show-overflow-tooltip>
                                    </el-table-column>
                                </el-table>
                            </el-main>
                        </template>
                    </el-main>
                </el-container>
            </el-container>
        </el-main>
    </el-container>
</template>

<script>
// 导入所需的模块和组件
import axios from 'axios';
import { list_compare_history, delete_clause_his, compare_clause, upload_new_clause, list_standard_clause, clause_doc_stream_check, delete_dialogue, chatclauseStreamgpt, getclausehistory, getChatMsg, chatgpt, chatupload, gethistory, setclause_check, getChat, getChatchat, getclauseChat } from "@/api/getData";
import Emoji from "@/components/Emoji.vue";
import Nav from "@/components/Nav.vue";
import commonMethodsMixin from '../../util/publicfun.js';
import StreamText from '@/components/StreamText.vue';
import VueMarkdown from 'vue-markdown';

export default {
    // 混入公共方法
    mixins: [commonMethodsMixin],
    
    // 注册使用的组件
    components: {
        Emoji,
        Nav,
        StreamText,
        VueMarkdown
    },
    
    // 生命周期钩子 - 组件创建时
    created() {
        // 定义请求参数
        let params = {
            token: "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJuYW1lIjoidGVzdDEiLCJleHAiOjE3Mjg4Nzc1MDl9.10pwn0YnmSqIe7Ixsfozf1wDbk7RF4dn4KKc1NQWe7g"
        }
        console.log("params", params)
        
        // 获取标准条款列表
        list_standard_clause(params).then((res) => {
            if (res.data.code == 200) {
                this.options = this.uniqueByField(res.data.data, 'file_id').map(item => ({
                    value: item.file_id,
                    label: item.clause_name
                }))
                this.value = this.options.length > 0 ? this.options[0].value : ''
            } else {
                this.$message.error('获取标准条款列表失败!');
            }
        }).catch((err) => {
            this.$message.error('获取标准条款列表失败!');
        })
        
        // 获取比对历史记录
        list_compare_history(params).then((res) => {
            console.log("list_compare_history", res.data.data)
            this.historyArrlist = res.data.data
        })
    },
    
    
    // 组件数据
    data() {
        return {
            // 标准条款下拉框
            options: [{
                value: '选项1',
                label: '安全培训需求调查'
            }, {
                value: '选项2',
                label: '年度安全培训计划制定'
            }, {
                value: '选项3',
                label: '安全生产教育培训制度'
            }],
            // 标准条款下拉框选中值 
            value: '选项1',
            // 比对结果表格数据
            clauseTableData: [
                {
                    index: 1,
                    id: '4b370157-896b-11ef-ac83-2cf05d3470d1',
                    institution: '安全培训需求调查',
                    clause_name: '安全生产教育培训制度',
                    clause_number: '《国务院安委会关于进一步加强安全培训工作的决定》（安委[2012]10号）第二十条',
                    original_text:
                        '加强安全培训过程管理和质量评估。建立安全培训需求调研、培训策划、培训计划备案、教学管理、培训效果评估等制度，加强安全培训全过程管理。',
                    related_chunks:
                    {
                        file_id: "259dfc41-896c-11ef-91da-2cf05d3470d1",
                        file_name: "03 测试制度1-安全培训教育制度.docx",
                        content: "培训计划和培训资金 安全部负责开展安全培训需求调研工作，各职能部门根据本部门的需要，在年初提交培训需求给安全部，由安全部统一汇总。 安全部汇总安全培训需求，确定安全培训所需费用，根据培训需求和国家相关规定，制定公司年度安全培训计划，提交总经理批准。 总经理批准后，安全部按照年度培训计划组织开展安全培训。 任何部门不得挪用安全培训资金。",
                    },
                    review_result: '符合',
                    advice: '无',
                },
                {
                    index: 2,
                    id: '4b3c3208-896b-11ef-8ebe-2cf05d3470d1',
                    institution: '年度安全培训计划制定',
                    clause_name: '安全生产教育培训制度',
                    clause_number: '《生产经营单位安全培训规定》（原国家安全生产监督管理总局令第80号）第二十一条第一款',
                    related_chunks: {
                        file_id: "259dfc41-896c-11ef-91da-2cf05d3470d1",
                        file_name: "03 测试制度1-安全培训教育制度.docx",
                        content: "培训计划和培训资金 安全部负责开展安全培训需求调研工作，各职能部门根据本部门的需要，在年初提交培训需求给安全部，由安全部统一汇总。 安全部汇总安全培训需求，确定安全培训所需费用，根据培训需求和国家相关规定，制定公司年度安全培训计划，提交总经理批准。 总经理批准后，安全部按照年度培训计划组织开展安全培训。 任何部门不得挪用安全培训资金。",
                    },
                    original_text: '生产经营单位应当将安全培训工作纳入本单位年度工作计划。保证本单位安全培训工作所需资金。',
                    review_result: '符合',
                    advice: '无',
                },
                {
                    index: 3,
                    id: '4bef97cb-896b-11ef-93a7-2cf05d3470d1',
                    institution: '安全培训教育经费',
                    clause_name: '安全生产教育培训制度',
                    clause_number: '《生产经营单位安全培训规定》（原国家安全生产监督管理总局令第80号） 第二十一条',
                    original_text: '生产经营单位应当将安全培训工作纳入本单位年度工作计划。保证本单位安全培训工作所需资金。',
                    related_chunks: { 
                        file_id: "259dfc41-896c-11ef-91da-2cf05d3470d1",
                        file_name: "03 测试制度1-安全培训教育制度.docx",
                        content: "培训计划和培训资金 安全部负责开展安全培训需求调研工作，各职能部门根据本部门的需要，在年初提交培训需求给安全部，由安全部统一汇总。 安全部汇总安全培训需求，确定全培训所需费用，根据培训需求和国家相关规定，制定公司年度安全培训计划，提交总经理批准。总经理批准后，安全部按照年度培训计划组织开展安全培训。 任何部门不得挪用安全培训资金。",
                    },
                    review_result: '符合',
                    advice: '无',
                }],
            selected: "3",
            newMessage: "",
            promptall: [{
                value: "生成摘要",
            }, {
                value: '标签提取',
            }, {
                value: '内容检查',
            }],
            promptdefaultvalue: '',
            isCollapse: false,
            cards: [
                {
                    icon: 'el-icon-info',
                    title: '标题一',
                    subtitle: '副标题一'
                },
                {
                    icon: 'el-icon-warning',
                    title: '标题二',
                    subtitle: '副标题二'
                },
                {
                    icon: 'el-icon-error',
                    title: '标题三',
                    subtitle: '副标题三'
                },
                {
                    icon: 'el-icon-success',
                    title: '标题四',
                    subtitle: '副标题四'
                }
            ],
            fileList: [],
            chat_id: "",
            newhistory: {},
            /**
             * 历史记录列表
             * [{
             *  id: 对话id
             *  name: 对话名称
             *  showDeleteButton: 是否显示删除按钮,
             *  compare_result: 比对结果
             * }]
             * 
             * compare_result： [{
             *  id: 序号
             *  institution: 制度要素
             *  clause_number: 法规条款
             *  original_text: 法规原文
             *  related_chunks: 企业制度
             *  review_result: 检查结果
             *  advice: 修改建议
             * }]
            */
            historyArrlist: [],
            configs: [],
            promptall: [],
            promptdefaultvalue: '',
            dynamicMarginLeft: '50px',
            isCollapse: false,
            cards: [
                {
                    icon: 'el-icon-info',
                    title: '标题一',
                    subtitle: '副标题一'
                },
                {
                    icon: 'el-icon-warning',
                    title: '标题二',
                    subtitle: '副标题二'
                },
                {
                    icon: 'el-icon-error',
                    title: '标题三',
                    subtitle: '副标题三'
                },
                {
                    icon: 'el-icon-success',
                    title: '标题四',
                    subtitle: '副标题四'
                }
            ],
            cards1: [
                // {
                //     header: '翻译',
                //     content: '帮我翻译苹果',
                //     message: '帮我翻译苹果'
                // },
                // {
                //     header: '摘要',
                //     content: '帮我分析这个摘要',
                //     message: '帮我分析这个摘要'
                // },
                // {
                //     header: '试题生成',
                //     content: '帮我生成一个试题',
                //     message: '帮我生成一个试题'
                // }
            ],
            chatMessages: [],
            chatStarted: false,
            chatMessagesList: [],
            //上传后的文件列表
            fileList: [],
            // 允许的文件类型
            fileType: ["pdf", "doc", "docx", "xls", "xlsx", "txt", "png", "jpg", "bmp", "jpeg"],
            // 运行上传文件大小，单位 M
            fileSize: 50,
            // 附件数量限制
            fileLimit: 5,
            //请求头
            headers: { "Content-Type": "multipart/form-data" },
            isHovered: [],
            // 是否上传完成
            hasUploadDone: false,
            // 是否比对完成
            hasCompareDone: false, 
            // 步骤条【0：选择标准条款，1：上传待审核条款，2：新文档理解中，3：比对完成】
            activeStep: 0,
            abortController: null,
            customColors: [
                {color: '#a8d3ff', percentage: 20},
                {color: '#7ebcff', percentage: 40},
                {color: '#54a5ff', percentage: 60},
                {color: '#2a8eff', percentage: 80},
                {color: '#1385f6', percentage: 100}
            ]
        };
    },
    
    // 方法定义
    methods: {
        // 去重
        uniqueByField(arr, field) {
            const seen = new Set();
            return arr.filter(item => {
                const value = item[field];
                if (!seen.has(value)) {
                    seen.add(value);
                    return true;
                }
                return false;
            });
        },
        // 历史聊天记录选择
        historyChat(question, index) {
            console.log("this.question", question)
            // this.chatStarted = true;
            // this.historyArrlist.forEach((item, i) => {
            //     item.showDeleteButton = i === index;
            // });
            // this.newhistory = question
            // this.chatMessages = question.history
            // this.chat_id = question.dialogue_id
        },
        
        // 滚动到底部
        scrollToBottom() {
            this.$nextTick(() => {
                const container = this.$refs.chatContainer;
                if (container) {
                    container.scrollTop = container.scrollHeight;
                }
            });
        },
        
        // 显示完整引用
        showFullReference(messageIndex, referenceIndex) {
            console.log("this.chatMessages[messageIndex]", this.chatMessages[messageIndex])
            this.$set(this.chatMessages[messageIndex].isHovered, referenceIndex, true);
        },
        
        // 隐藏完整引用
        hideFullReference(messageIndex, referenceIndex) {
            this.$set(this.chatMessages[messageIndex].isHovered, referenceIndex, false);
        },
        
        // 显示删除按钮
        showDeleteButton(index) {
            this.$set(this.historyArrlist[index], 'showDeleteButton', true);
        },
        
        // 隐藏删除按钮
        hideDeleteButton(index) {
            this.$set(this.historyArrlist[index], 'showDeleteButton', false);
        },
        
        // 删除项目
        deleteItem(index) {
            console.log("deleteItem", this.historyArrlist[index])

            let params = {
                his_id: this.historyArrlist[index].id,
                token: "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJuYW1lIjoidGVzdDEiLCJleHAiOjE3Mjg4Nzc1MDl9.10pwn0YnmSqIe7Ixsfozf1wDbk7RF4dn4KKc1NQWe7g"
            }
            this.$confirm('此操作将永久删除该对话, 是否继续?', '提示', {
                confirmButtonText: '确定',
                cancelButtonText: '取消',
                type: 'warning'
            }).then(() => {

                delete_clause_his(params).then((res) => {
                    this.$message({
                        type: 'success',
                        message: '删除成功!'
                    });
                    this.historyArrlist.splice(index, 1);
                    if (this.activeIndex === index) {
                        this.activeIndex = 0;
                        this.clauseTableData = [];
                        this.activeStep = 1;
                    } else if (this.activeIndex > index) {
                        this.activeIndex--;
                    }
                    // this.$nextTick(() => {
                    //     document.activeElement.blur();
                    // });
                    // this.$nextTick(() => {
                    //     this.$refs.dummyInput.focus();
                    // });
                }).catch((err) => {

                })

            }).catch(() => {
                this.$message({
                    type: 'info',
                    message: '已取消删除'
                });
            });

        },
        
        // 处理选择
        handleSelect(index) {
            this.activeIndex = Number(index);
            // 处理选中项逻辑
            this.clauseTableData = this.historyArrlist[index].compare_result
            this.activeStep = 4;
            this.hasCompareDone = true;
            this.scrollTableToTop();
        },
        
        // 获取用户内容
        getUserContent(history) {
            return history.name
            // if (history != null) {
            //     const userEntry = history.find(entry => entry.role !== 'assistant');
            //     return userEntry ? userEntry.name : '';
            // }
            // else {
            //     return ""
            // }

        },
        
        // 更新折叠状态
        updateIsCollapse(value) {
            this.isCollapse = value;
            // this.updateIsCollapse(value);
        },
        
        // 上传前检查
        beforeUpload(file) {
            const FileExt = file.name.split('.').pop().toLowerCase();
            const isLt50M = file.size / 1024 / 1024 < 50;

            if (!isLt50M) {
                this.$message.error('上传文件大小不能超过 50MB!');
                return false;
            }

            if (!this.fileType.includes(FileExt)) {
                this.$message.error('上传文件格式不正确!');
                return false;
            }

            return true;
        },
        
        // 处理关闭
        handleClose(index) {
            this.fileList.splice(index, 1);
            if (this.fileList.length === 0) {
                this.fileflag = true;
                this.$set(this.rules.url, 0, { required: true, validator: this.validatorUrl, trigger: 'blur' });
            }
        },
        
        // 处理超出限制
        handleExceed() {
            this.$message({
                type: 'warning',
                message: '超出最大上传文件数量的限制！'
            });
            return;
        },
        
        // 上传文件
        async uploadFile(item) {
            //上传文件的需要formdata类型;所以要转
            var FormDatas = new FormData()
            FormDatas.append('file', item.file);

            this.dialogFileUrl = item.file.name;
            let params = {
                token: "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJuYW1lIjoidGVzdDEiLCJleHAiOjE3Mjg4Nzc1MDl9.10pwn0YnmSqIe7Ixsfozf1wDbk7RF4dn4KKc1NQWe7g",
                file: item.file
            }
            this.fileList.push(item.file);
            upload_new_clause(params).then(async res => {
                console.log("res-上传", res)
                if (res.data.code == 200) {
                    this.hasUploadDone = true
                    // 睡眠1s
                    await new Promise(resolve => setTimeout(resolve, 1000));
                    this.activeStep = 2
                    // 创建新的 AbortController
                    this.abortController = new AbortController();
                    const compareparams = {
                        token: "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJuYW1lIjoidGVzdDEiLCJleHAiOjE3Mjg4Nzc1MDl9.10pwn0YnmSqIe7Ixsfozf1wDbk7RF4dn4KKc1NQWe7g",
                        stand_id: this.value,
                        new_id: res.data.data.id
                    };
                    compare_clause(compareparams, this.handleChunk1, this.abortController.signal);
                    // 睡眠1s
                    await new Promise(resolve => setTimeout(resolve, 1000));
                    this.activeStep = 3
                } else {
                    this.$message.error('上传失败!');
                }
            }).catch((err) => {
                console.log(err.response.data.msg)
                this.$message.error('上传失败!, 错误信息：' + err.response.data.msg);
            })
        },
        
        // 处理上传成功
        handleSuccess() {

        },
        
        // 移除前处理
        beforeRemove() {

        },
        
        // 处理预览
        handlePreview() {

        },
        
        // 生成GUID
        guid() {
            return 'xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx'.replace(/[xy]/g, function (c) {
                var r = Math.random() * 16 | 0,
                    v = c == 'x' ? r : (r & 0x3 | 0x8);
                return v.toString(16);
            });
        },
        
        // 开始聊天
        startChat() {
            console.log("this.chat_id", this.chat_id)

            if (this.newMessage.trim() !== '' || this.newMessage.trim().length > 0 || this.fileList.length > 0) {
                if (this.fileList.length > 0) {
                    // this.newMessage = ""
                    this.chatMessages.push({ content: "让我分析这个文档      " + this.fileList[0].name, role: 'user' });

                }
                else {
                    this.chatMessages.push({ content: this.newMessage, role: 'user' });
                }
                if (this.chat_id == "") {
                    this.chat_id = this.guid()
                }
                console.log("chat_id", this.chat_id)
                let config = {
                    "model": "deepseek",
                    "prompt": "default",
                    "knowledge": "default",
                    "LLM_config": "default"
                }
                let params = {
                    dialogue_id: this.chat_id,
                    query: this.newMessage,
                    config: JSON.stringify(config)
                    // history: JSON.stringify([{role:"hh",content:"xx"},{role:"hh",content:"xx"}])
                    // {role:"hh",content:"xx"}
                    // ,
                }
                console.log("params", params)
                if (this.fileList.length > 0) {
                    let params1 = {
                        dialogue_id: this.chat_id,
                        config: JSON.stringify(config),
                        file: this.fileList[0]
                    }
                    clause_doc_stream_check(params1, this.handleChunk1)
                }
                else {
                    chatclauseStreamgpt(params, this.handleChunk, this.handleReferences);

                }
            }
            else {


                this.$alert('请输入内容', '提示', {
                    confirmButtonText: '确定',
                    callback: action => {

                    }
                });


            }


        },
        
        // 处理数据块
        handleChunk(first, content) {
            if (first) {
                this.chatMessages.push({ content: '', role: 'assistant', reference: [] });
            }
            console.log("content", content)

            const lastMessageIndex = this.chatMessages.length - 1;
            this.chatMessages[lastMessageIndex].content += content;
            this.$set(this.chatMessages, lastMessageIndex, { ...this.chatMessages[lastMessageIndex] });
            this.newhistory = {
                dialogue_id: this.chat_id, history: this.chatMessages
            }
            const existingIndex = this.historyArrlist.findIndex(item => item.dialogue_id === this.chat_id);

            if (existingIndex === -1) {
                this.historyArrlist.unshift(this.newhistory);
            } else {
                console.log("Duplicate dialogue_id found, not adding.");
            }
            this.newMessage = '';
            this.chatStarted = true;
        },
        
        // 处理引用
        handleReferences(reference) {
            console.log("this.chatMessages111", this.chatMessages);

            if (this.chatMessages.length === 0 || !this.chatStarted) {
                // If no message exists, create a new one
                this.chatMessages.push({ content: '', role: 'assistant', reference: [] });
                this.newMessage = '';
                this.chatStarted = true;
            }
            const lastMessageIndex = this.chatMessages.length - 1;
            this.chatMessages[lastMessageIndex].reference = JSON.parse(reference).reference;
            this.chatMessages[lastMessageIndex].content = JSON.parse(reference).response;

            this.$set(this.chatMessages, lastMessageIndex, { ...this.chatMessages[lastMessageIndex] });

            // Initialize itemDetailsVisible array
            this.itemDetailsVisible = Array(this.chatMessages[lastMessageIndex].reference.length).fill(false);

            console.log("this.chatMessages", this.chatMessages);

            // Update the historyArrlist
            this.newhistory = {
                dialogue_id: this.chat_id, history: this.chatMessages
            };

            const existingIndex = this.historyArrlist.findIndex(item => item.dialogue_id === this.chat_id);

            if (existingIndex === -1) {
                this.historyArrlist.unshift(this.newhistory);
            } else {
                this.historyArrlist[existingIndex] = this.newhistory;
            }
        },
        
        // 新建聊天
        newChat() {
            this.activeStep = 1
            this.hasCompareDone = false
        },
        
        // 发送消息
        sendMessage(text) {
            this.newMessage = text;
        },
        
        // 切换折叠状态
        toggleCollapse() {
            this.isCollapse = !this.isCollapse; // 切换状态
        },
        
        // 处理打开
        handleOpen(key, keyPath) {
            console.log(key, keyPath);
        },
        
        // 处理关闭
        handleClose(key, keyPath) {
            console.log(key, keyPath);
        },
        
        // 页面跳转方法
        goToMain() {
            window.location.href = '#/ChatHome';
        },
        goToKnowledgeQA() {
            window.location.href = '#/KnowLedgeChat';
        },
        goToFreeChat() {
            window.location.href = '#/FreeChat';
        },
        goToCheckChat() {
            window.location.href = '#/CheckChat';
        },
        goToTitleSetChat() {
            window.location.href = '#/TitleSetChat';
        },
        goToKnowSetting() {
            window.location.href = '#/KnowSetting';
        },
        goToPrompt() {
            window.location.href = '#/Prompt';
        },
        goToSelectModel() {
            window.location.href = '#/SelectModel';
        },
        goToHelp() {
            window.location.href = '#/HelpChat';
        },
        
        // 移除文件
        removeFile(index) {
            this.fileList = []
        },
        // 添加新方法
        goToHome() {
            this.$router.push('/ChatHome');
        },
        // 滚动到顶部
        scrollTableToTop() {
            this.$nextTick(() => {
                if (this.$refs.clauseTable) {
                    const tableBody = this.$refs.clauseTable.$el.querySelector('.el-table__body-wrapper');
                    if (tableBody) {
                        tableBody.scrollTop = 0;
                    }
                }
            });
        },
        // 滚动到底部
        scrollTableToBottom() {
            this.$nextTick(() => {
                if (this.$refs.clauseTable) {
                    const tableBody = this.$refs.clauseTable.$el.querySelector('.el-table__body-wrapper');
                    if (tableBody) {
                        tableBody.scrollTop = tableBody.scrollHeight;
                    }
                }
            });
        },
        // 修改 handleChunk1 方法
        handleChunk1(first, content, end) {
            if (first) {
                this.activeStep = 4
                this.clauseTableData = []
                // 构建新对比历史
                const newHistory = {
                    name: this.options.filter(item => item.value === this.value)[0].label,
                    compare_result: []
                }
                this.historyArrlist.unshift(newHistory)
            }
            // 结束
            if (end) {
                console.log("比对完成")
                this.hasCompareDone = true
                return
            }
            // 比对内容插入
            if (content) {
                try {
                    const jsonData = JSON.parse(content);
                    this.clauseTableData.push(jsonData);
                    this.historyArrlist[0].compare_result.push(jsonData)
                    this.scrollTableToBottom(); // 每次添加新数据后滚动到底部
                } catch (error) {
                    console.error('Error parsing JSON:', error);
                }
            }
        },
    },
    beforeRouteLeave(to, from, next) {
        // 在路由离开前中断连接
        if (this.abortController) {
            this.abortController.abort();
        }
        next();
    },
    beforeDestroy() {
        // 组件销毁前中断连接
        if (this.abortController) {
            this.abortController.abort();
        }
    },
}
</script>

<style scoped lang="scss">
.el-menu .el-menu-item {
    width: 100%;
}

.uploadcontainer {
    display: flex;
    align-items: center;
}

.upload-wrapper {
    margin-right: 10px;
    /* 调整上传按钮和输入框之间的间距 */
    flex-grow: 1;
    /* 占据剩余空间 */
}

.input-wrapper {
    display: flex;
    align-items: center;
    background-color: #f1f1f1;
    border-radius: 20px;
    width: 100%;
    height: 40px;
    justify-content: space-between;
}

.input-button {
    display: flex;
    align-items: center;
    background-color: transparent;
    justify-content: space-around;
    flex-direction: row;
}

.input-field-wrapper {
    display: flex;
    align-items: center;
    flex-grow: 1;
    position: relative;
}

.input-field .el-input__inner {
    border: none;
    background: transparent;
    box-shadow: none;
    /* flex-grow: 1; */
}

.file-tag {
    display: flex;
    align-items: center;
    background-color: #e0e0e0;
    border-radius: 12px;
    padding: 4px 4px;
    position: relative;
    line-height: 18px;
}

.file-tag span {
    margin-right: 0px;
}

.file-tag i {
    cursor: pointer;
    display: none;
}

.file-tag:hover i {
    display: inline-block;
}

.menu-item-history:hover {
    border-radius: 10px;
    /* 圆角 */
}

.ellipsis {
    overflow: hidden;
    white-space: nowrap;
    text-overflow: ellipsis;
    max-width: 300px;
    /* 确保不会超出父容器 */
    padding: 5px;
    /* 内边距 */
    border-radius: 4px;
    /* 圆角 */
    transition: background-color 0.3s;
    /* 动画效果 */
}


.custom-tooltip {
    max-width: 300px;
    /* 设置最大宽度 */
    word-wrap: break-word;
    /* 自动换行 */
}



  /* 修改完成状态的标题字体颜色 */
  :deep(.el-step__title.is-success)  {
    color: #1385f6 !important;/* 使用 Element UI 的成功绿色或其他您喜欢的颜色 */
  }

  /* 修改图标颜色 */
  :deep(.el-step__head.is-success) {
    .el-step__icon.is-text {
      border-color: #1385f6 !important;
      color: #1385f6 !important;
    }
    .el-step__line {
      background-color: #1385f6 !important;
      color: #1385f6 !important;
      border-color: #1385f6 !important;
    }
  }

  ::v-deep .el-progress-bar__inner {
    transition: width 0.6s ease;
  }

  ::v-deep .el-progress__text {
    color: #1385f6;
  }
</style>
<style>
  .el-tooltip__popper {
    max-width: 800px;
  }
</style>
