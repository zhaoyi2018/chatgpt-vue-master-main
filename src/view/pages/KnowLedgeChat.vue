<template>
    <el-container class="background-container">
        <!-- 页面头部 -->
        <el-header class="header-container" style="height: 60px;">
            <img src="../../imgs/logo1.png" class="logo" />

        </el-header>
        <!-- 页面主体 -->
        <el-main style="padding: 0px;">
            <el-container class="main-container">
                <!-- 左侧边栏 -->
                <el-aside width="220px" class="aside-container">
                    <el-header class="aside-header">
                        <el-button type="primary" icon="el-icon-plus" @click="newChat" round>新对话</el-button>
                    </el-header>
                    <el-menu @select="handleSelect" :default-active="activeIndex" class="custom-scrollbar"
                        style="background-color: #f2fbff; justify-content: center;  height:70vh; margin-bottom:20px; overflow-x: hidden; ">
                        <el-menu-item v-for="(question, index) in historyArrlist" :key="index" :index="index.toString()"
                            class="menu-item-history" @click="listConversion(index)">
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
                    <!-- 聊天内容显示区 -->
                    <el-main class="main-content">
                        <!-- 聊天内容显示区 -->
                        <div v-if="chatStarted" class="chat-container" ref="chatContainer">
                            <div v-for="(message, index) in chatMessages" :key="index" class="chat-message">
                                <div v-if="message.role === 'user'" class="answer-message">
                                    <div class="card">
                                        <i class="el-icon-user" style="line-height: 1.5;"> {{ message.content }}</i>
                                    </div>
                                </div>
                                <div v-else-if="message.role === 'assistant'" class="answer-message">
                                    <div class="card" style=" background-color: #fff; float: left; color:#000; ">
                                        <div v-if="message.reference.length > 0">

                                            <i class="el-icon-paperclip"
                                                style="margin-top: 10px;margin-bottom: 10px;">Reference</i>

                                            <div v-for="(item, index1) in message.reference" :key="index1"
                                                class="reference-item">
                                                <!-- 对内容进行截取显示： 1，添加序号 2，截取长度 -->
                                                <div class="reference-content" @click="showFullContent(item)">
                                                    <!-- 添加序号 -->
                                                    <span class="reference-number">{{ index1 + 1 }}. </span>
                                                    <!-- 显示标准名称,条款,编号 -->
                                                    <el-tag v-if="item.standard_name" size="mini">{{ item.standard_name }}-{{ item.standard_clause }}-{{ item.standard_number }}</el-tag>
                                                    <!-- 表名 -->
                                                    <el-tag v-else size="mini">{{ item.table }}</el-tag>
                                                    <!-- 截取内容 -->
                                                    <span class="reference-text">
                                                        {{ truncateText(item.content, 20) }}
                                                    </span>
                                                    
                                                </div>
                                            </div>
                                            <el-divider></el-divider>
                                        </div>
                                        <span v-if="index === chatMessages.length - 1">
                                            <vue-markdown :source="message.content" :breaks="true" :typographer="true"
                                                :linkify="true" :highlight="false"></vue-markdown>
                                        </span>
                                        <span v-else>
                                            <vue-markdown :source="message.content" :breaks="true" :typographer="true"
                                                :linkify="true" :highlight="false"></vue-markdown>
                                        </span>



                                    </div>
                                </div>
                            </div>
                        </div>
                    </el-main>
                    <!-- 聊天输入区 -->
                    <el-footer style="align-items: flex-start; display: flex;margin-bottom: 5px; padding: 0px;">
                        <div class="input-wrapper">
                            <!-- 输入框 -->
                            <div class="input-field-wrapper">
                                <el-input v-model="newMessage" class="input-field" placeholder="请输入内容"
                                    @input="sendMessage" @keyup.enter.native="startChat">
                                </el-input>

                            </div>
                            <!-- 发送按钮 -->
                            <div class="input-button">
                                <i class="el-icon-s-promotion" @click="startChat" style="margin-right: 5px;">
                                </i>
                            </div>

                        </div>
                    </el-footer>
                </el-container>
            </el-container>
        </el-main>
        <!-- 相关内容弹窗组件， 居中显示 -->
        <el-dialog
            :visible.sync="dialogVisible"
            :title="fullReference"
            width="50%"
            align-center
        >
            <vue-markdown :source="fullContent" @rendered="addClickEvents"></vue-markdown>
        </el-dialog>
        <!-- 添加预览页面的对话框 -->
        <el-dialog
            :visible.sync="previewDialogVisible"
            title="图片预览"
            width="50%"
            append-to-body
            align-center
        >
            <!-- 图片预览区 -->
            <div class="image-container">
                <img v-if="previewUrl !== '无图片'" :src="previewUrl" alt="预览图片" class="preview-image"/>
                <div v-else class="no-image-placeholder">
                    <i class="el-icon-picture-outline"></i>
                    <p style="display: inline-block;">无法加载图片</p>
                </div>
            </div>
        </el-dialog>
    </el-container>
</template>

<script>
import { chatkbStreamgpt, updateDialog, chatupload, getChat, delete_dialogue, getkbChat, login, list_conversion } from "@/api/getData";
import Emoji from "@/components/Emoji.vue";
import Nav from "@/components/Nav.vue";
import commonMethodsMixin from '../../util/publicfun.js';
import StreamText from '@/components/StreamText.vue';
import VueMarkdown from 'vue-markdown';

export default {
    mixins: [commonMethodsMixin],
    components: {
        Emoji,
        Nav,
        StreamText,
        VueMarkdown
    },
    data() {
        return {
            // 默认知识库id
            defaultKbId: ['89bef038-8132-11ef-9800-2cf05d3470d1', '969c9f22-8132-11ef-8470-2cf05d3470d1', 'cae9d17d-8092-11ef-a840-2cf05d3470d1'],
            // 当前回答内容
            currentStreamMessage: '',
            // 左侧激活对话索引(从0开始)
            activeIndex: null,
            selected: '1',
            // 对话id
            dialogue_id: "",
            /**
             * 当前对话item简介
             * {id,name，kb_ids, dialog_type}
             * */ 
            newhistory: {},
            /**
             * 历史对话框列表
             * [{id,name，kb_ids, dialog_type}]
            */
            historyArrlist: [],
            promptall: [],
            models: [],
            promptdefaultvalue: 'default',
            dynamicMarginLeft: '50px',
            isCollapse: false,
            // 输入框内容（用户问话内容）
            newMessage: '',
            /**
             * 聊天消息（包括用户问话和回答）
             * [{
             * role: "角色", 
             * content: "问话内容或回答内容", 
             * reference: [{标准名称,条款,编号,内容,图片}]
             * }]
             * 
            */
            chatMessages: [],
            // 聊天是否开始
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
            itemDetailsVisible: [],
            isHovered: [],
            // 弹窗组件
            dialogVisible: false,
            fullContent: '',
            fullReference: '',
            previewDialogVisible: false,
            previewUrl: "无图片"
        };
    },
    created() {
        var token = ""
        // 获取token
        login().then((res) =>   {
            // 获取token
            token = res.data.data.access_token
            var params = {
                access_token: token
            }
            // 获取对话历史列表
            getChat(params).then((res) => {
                // 设置对话id
                this.dialogue_id = res.id
                // 聊天开始
                this.chatStarted = true;
                // 设置历史记录
                this.historyArrlist = res.data
            }).catch((err) => {
                console.log("err")
            })
        })
    },
    watch: {
        chatMessages: {
            handler(newMessages) {
                this.scrollToBottom();
                newMessages.forEach(message => {
                    // 确保 message 对象具有 reference 属性并进行初始化
                    if (!message.reference) {
                        this.$set(message, 'reference', []);
                    }
                    if (!message.isHovered) {
                        this.$set(message, 'isHovered', Array(message.reference.length).fill(false));
                    }
                });
            },
            immediate: true,
            deep: true
        }
    },
    methods: {
        // 截取内容
        truncateText(text, maxLength) {
            if (text.length <= maxLength) {
                return text;
            }
            return text.slice(0, maxLength) + '...';
        },
        // 显示完整内容
        showFullContent(item) {
            if (item.standard_name) {
                this.fullReference = item.standard_name + "-" + item.standard_clause + "-" + item.standard_number;
                // item.image = 'd13ed1d0-8afe-11ef-98cc-00163e1e6539'
                this.fullContent = this.convertBracketsToSpans(item.content, item.image ? `http://121.43.126.21:8001/image/${item.image}` : "无图片");
            } else {
                this.fullReference = item.table;
                this.fullContent = this.appendCustomLink(item.content, item.table, item.image ? `http://121.43.126.21:8001/image/${item.image}` : "无图片");
            }
            this.dialogVisible = true;
            
        },
        // 追加自定义链接
        appendCustomLink(text, title, url) {
            return `<span class="clickable-text" style="font-weight: bolder;cursor: pointer;color: blue;" data-text="${title}" data-url="${url}" >${title}</span>\n\n${text}`;
        },
        // 将文本内【】标记的内容，转为VueMarkdown可识别的span结构
        convertBracketsToSpans(text, url) {
            const regex = /【(.+?)】/g;
            return text.replace(regex, (match, p1) => {
                return `<span class="clickable-text" style="font-weight: bolder;cursor: pointer;color: blue;s" data-text="${p1}" data-url="${url}" >【${p1}】</span>`;
            });
        },
        // 添加点击事件
        addClickEvents() {
            this.$nextTick(() => {
                const clickableTexts = document.querySelectorAll('.clickable-text');
                clickableTexts.forEach(element => {
                    element.addEventListener('click', this.handleClickableTextClick);
                });
            });
        },
        // 点击文本事件
        handleClickableTextClick(event) {
            const text = event.target.dataset.text;
            const url = event.target.dataset.url;
            this.previewUrl = url;
            this.previewDialogVisible = true;
        },
        showFullReference(messageIndex, referenceIndex) {
            console.log("this.chatMessages[messageIndex]", this.chatMessages[messageIndex])
            this.$set(this.chatMessages[messageIndex].isHovered, referenceIndex, true);
        },
        hideFullReference(messageIndex, referenceIndex) {
            this.$set(this.chatMessages[messageIndex].isHovered, referenceIndex, false);
        },
        handleSelectDrop(item) {
            console.log("promptdefaultvalue", item.value)
            this.promptdefaultvalue = item.value
        },
        scrollToBottom() {
            this.$nextTick(() => {
                const container = this.$refs.chatContainer;
                if (container) {
                    container.scrollTop = container.scrollHeight;
                }
            });
        },
        getUserContent(history) {
            if (history.length > 0) {
                const userEntry = history.find(entry => entry.role === 'user');
                return userEntry ? userEntry.content : '';
            }
            else {
                return ""
            }

        },
        showDeleteButton(index) {
            this.$set(this.historyArrlist[index], 'showDeleteButton', true);
        },
        hideDeleteButton(index) {
            this.$set(this.historyArrlist[index], 'showDeleteButton', false);
        },
        listConversion(index) {
            let params = {
                id: this.historyArrlist[index].id,
                token: "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJuYW1lIjoidGVzdDEiLCJleHAiOjE3Mjg4Nzc1MDl9.10pwn0YnmSqIe7Ixsfozf1wDbk7RF4dn4KKc1NQWe7g"
            }
            list_conversion(params).then((res) => {
                // 设置对话id
                this.dialogue_id = this.historyArrlist[index].id
                // 聊天开始
                this.chatStarted = true;
                // 设置聊天消息
                this.chatMessages = []
                // 循环设置聊天消息 
                res.data.data.forEach(item => {
                    // 设置用户问话 
                    this.chatMessages.push({ content: item.question, role: 'user' });
                    // 设置回答
                    this.chatMessages.push({ content: item.answer, role: 'assistant', reference: item.reference });
                })
            }).catch((err) => {

            })
        },
        deleteItem(index) {
            let params = {
                dialog_id: this.historyArrlist[index].id,
                token: "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJuYW1lIjoidGVzdDEiLCJleHAiOjE3Mjg4Nzc1MDl9.10pwn0YnmSqIe7Ixsfozf1wDbk7RF4dn4KKc1NQWe7g",
            }
            this.$confirm('此操作将永久删除该对话, 是否继续?', '提示', {
                confirmButtonText: '确定',
                cancelButtonText: '取消',
                type: 'warning'
            }).then(() => {
                delete_dialogue(params).then((res) => {
                    // 判断是否删除成功 
                    if (res.data.code !== 200) {
                        this.$message({
                            type: 'error',
                            message: '删除失败!'
                        });
                    } else {
                        this.$message({
                            type: 'success',
                            message: '删除成功!'
                        });
                        // 删除历史记录
                        this.historyArrlist.splice(index, 1);
                        // 输入框内容初始化
                        this.newMessage = ""
                        // 聊天结束
                        this.chatStarted = false
                        // 移除对不存在元素的引用
                        this.$nextTick(() => {
                            if (this.activeIndex != null) {
                                if (parseInt(this.activeIndex) > index) {   
                                this.activeIndex = (parseInt(this.activeIndex) - 1).toString()
                                } else if (parseInt(this.activeIndex) == index) {
                                    this.activeIndex = null
                                    this.dialogue_id = ""
                                }
                            }   
                            document.activeElement.blur();
                        });
                    }
                }).catch((err) => {

                })

            }).catch(() => {
                this.$message({
                    type: 'info',
                    message: '已取消删除'
                });
            });

        },
        handleSelect(index) {
            this.activeIndex = index.toString();
            // 处理选中项逻辑
        },
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
        submitForm() {
            console.log('Submitting form data along with the fileList:',);

        },
        sendMessage(text) {
            this.newMessage = text;
        },
        guid() {
            return 'xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx'.replace(/[xy]/g, function (c) {
                var r = Math.random() * 16 | 0,
                    v = c == 'x' ? r : (r & 0x3 | 0x8);
                return v.toString(16);
            });
        },
        // 发送消息事件
        async startChat() {
            // 判断输入框内容是否为空
            if (this.newMessage.trim() !== '' || this.newMessage.trim().length > 0) {
                // 判断对话id是否为空
                if (this.dialogue_id == "") {
                    // 清空聊天消息
                    this.chatMessages = []
                    // 新建对话
                    this.newChat()
                    await new Promise(resolve => setTimeout(resolve, 2000));
                }
                // 添加聊天消息 
                this.chatMessages.push({ content: this.newMessage, role: 'user' });
                // 聊天参数
                let params = {
                    dialog_id: this.dialogue_id,
                    query: this.newMessage,
                    token: "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJuYW1lIjoidGVzdDEiLCJleHAiOjE3Mjg4Nzc1MDl9.10pwn0YnmSqIe7Ixsfozf1wDbk7RF4dn4KKc1NQWe7g",
                }
                // 发送消息
                chatkbStreamgpt(params, this.handleChunk, this.handleReferences);
                

            } else {
                // 提示输入内容
                this.$alert('请输入内容', '提示', {
                    confirmButtonText: '确定',
                    callback: action => {}
                });
            }
        },
        /**
         * 接收消息事件函数
         * @param {boolean} first 是否是第一条消息
         * @param {string} content 消息内容
         * @param {string} reference 参考内容
         * @param {boolean} end 是否是最后一条消息
         * */ 
        handleChunk(first, content, reference, end) {
            // 判断是否是第一条消息
            if (first) {
                if (this.chatMessages.length == 1) {    
                    // 构建当前会话框简介
                    this.historyArrlist[parseInt(this.activeIndex)].name = this.newMessage
                    this.historyArrlist[parseInt(this.activeIndex)].update_time = new Date().toISOString()
                    // 更新会话名称
                    const params = this.historyArrlist[parseInt(this.activeIndex)]
                    params.token = "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJuYW1lIjoidGVzdDEiLCJleHAiOjE3Mjg4Nzc1MDl9.10pwn0YnmSqIe7Ixsfozf1wDbk7RF4dn4KKc1NQWe7g"
                    params.dialog_id = params.id
                    this.updateDialogName(params)
                }
                // 添加回答
                this.chatMessages.push({ content: '', role: 'assistant', reference: JSON.parse(reference).reference });
                // 输入框内容初始化
                this.newMessage = '';
            }
            // 获取最后一条消息
            const lastMessageIndex = this.chatMessages.length - 1;
            // 判断是否是最后一条消息
            if (!end) {
                this.chatMessages[lastMessageIndex].content += content;
                this.$set(this.chatMessages, lastMessageIndex, { ...this.chatMessages[lastMessageIndex] });
            } else {
                this.chatMessages[lastMessageIndex].content = JSON.parse(content).response;
                this.$set(this.chatMessages, lastMessageIndex, { ...this.chatMessages[lastMessageIndex] });
                // 聊天开始
                this.chatStarted = true;
            } 
        },
        // 更新会话
        updateDialogName(params) {
            updateDialog(params).then((res) => {
                console.log("res", res)
            })
        },
        /**
         * 接收参考内容事件函数
         * @param {string} reference 参考内容
         * */ 
        handleReferences(reference) {
            console.log("reference", JSON.parse(reference).reference)
            // 获取最后一条消息
            const lastMessageIndex = this.chatMessages.length - 1;
            // 更新最后一条消息内容
            this.chatMessages[lastMessageIndex].content = JSON.parse(reference).response;

            this.chatMessages[lastMessageIndex].reference = JSON.parse(reference).reference;
            this.$set(this.chatMessages, lastMessageIndex, { ...this.chatMessages[lastMessageIndex] });
            console.log("this.chatMessages", this.chatMessages)
            // 初始化itemDetailsVisible数组
            this.itemDetailsVisible = Array(this.chatMessages[lastMessageIndex].reference.length).fill(false);

        },
        /**
         * 显示详情事件函数
         * @param {number} index 索引
         * */ 
        showDetails(index) {
            this.$set(this.itemDetailsVisible, index, true);
        },
        /**
         * 隐藏详情事件函数
         * @param {number} index 索引
         * */ 
        hideDetails(index) {
            this.$set(this.itemDetailsVisible, index, false);
        },
        /**
         * 新建对话事件函数
         * */ 
        newChat() {
            // 判断聊天消息是否为空
            if (this.chatMessages.length != 0) {
                // 设置聊天消息
                this.chatMessages = []
            }

             // 创建新会话
             var params = {
                token: "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJuYW1lIjoidGVzdDEiLCJleHAiOjE3Mjg4Nzc1MDl9.10pwn0YnmSqIe7Ixsfozf1wDbk7RF4dn4KKc1NQWe7g",
                kb_ids: this.defaultKbId,
                dialog_type: 1,
                name: "对话1"
            }
            getkbChat(params).then((res) => {
                // 对话id设置
                this.dialogue_id = res.data.dialog
                // 聊天开始
                this.chatStarted = true;
                // 创建新会话
                const newhistory = {
                    id: this.dialogue_id,
                    name: '新对话',
                    dialog_type: 1,
                    update_time: new Date().toISOString(),
                    kb_ids: this.defaultKbId,
                }
                // 添加新对话
                this.historyArrlist.unshift(newhistory);
                // 设置左侧激活对话索引
                this.activeIndex = '0'
            }).catch((err) => {
                console.log("err")
            })

        },
        historyChat(question, index) {
            // console.log("this.question", question)
            // this.chatStarted = true;
            // this.historyArrlist.forEach((item, i) => {
            //     item.showDeleteButton = i === index;
            // });
            // this.newhistory = question
            // this.chatMessages = question.history
            // this.dialogue_id = question.dialogue_id


        },


        //上传文件之前
        beforeUpload(file) {
            if (file.type != "" || file.type != null || file.type != undefined) {
                //截取文件的后缀，判断文件类型
                const FileExt = file.name.replace(/.+\./, "").toLowerCase();
                //计算文件的大小
                const isLt5M = file.size / 1024 / 1024 < 50; //这里做文件大小限制
                //如果大于50M
                if (!isLt5M) {
                    this.$showMessage('上传文件大小不能超过 50MB!');
                    return false;
                }
                //如果文件类型不在允许上传的范围内
                if (this.fileType.includes(FileExt)) {
                    return true;
                }
                else {
                    this.$message.error("上传文件格式不正确!");
                    return false;
                }
            }
        },
        //上传了的文件给移除的事件，由于我没有用到默认的展示，所以没有用到
        handleRemove() {
        },
        //这是我自定义的移除事件
        handleClose(i) {
            this.fileList.splice(i, 1);//删除上传的文件
            if (this.fileList.length == 0) {//如果删完了
                this.fileflag = true;//显示url必填的标识
                this.$set(this.rules.url, 0, { required: true, validator: this.validatorUrl, trigger: 'blur' })//然后动态的添加本地方法的校验规则
            }
        },
        //超出文件个数的回调
        handleExceed() {
            this.$message({
                type: 'warning',
                message: '超出最大上传文件数量的限制！'
            }); return
        },
        //上传文件的事件
        uploadFile(item) {
            // this.$showMessage('文件上传中........')
            //上传文件的需要formdata类型;所以要转
            console.log("FormDatas", item)

            var FormDatas = new FormData()
            FormDatas.append('file', item.file);
            console.log("FormDatas", FormDatas.get("file"))
            let params = {
                file: FormDatas.get("file"),
            }
            this.fileList.push(item.file);

            chatupload(params).then(res => {
                console.log("res", res.data.content)
                // if (res.data.id != '' || res.data.id != null) {
                //     this.fileList.push(item.file);//成功过后手动将文件添加到展示列表里
                //     let i = this.fileList.indexOf(item.file)
                //     this.fileList[i].id = res.data.id;//id也添加进去，最后整个大表单提交的时候需要的
                //     if (this.fileList.length > 0) {//如果上传了附件就把校验规则给干掉
                //         this.fileflag = false;
                //         this.$set(this.rules.url, 0, '')
                //     }
                //     //this.handleSuccess();
                // }
            })
        },
        //上传成功后的回调
        handleSuccess() {

        },
        beforeRemove() {

        },
        handlePreview() {

        },

    }
}
</script>

<style>
.background-container {
    background-image: url(../../imgs/bg.png);
    background-size: cover;
    /* Ensure the image covers the entire container */
    background-position: center;
    /* Center the image */
    background-repeat: no-repeat;
    /* Prevent the image from repeating */
    height: 100vh;
    /* Full viewport height */
    width: 100%;
    padding: 2px;
    /* Full width */
}

.header-container {
    display: flex;
    align-items: center;
    padding: 10px 20px;
    margin-left: 50px;
    height: 40px;
    justify-content: center;

    /* border-bottom: 1px solid #e0e0e0; */
}

.logo {
    width: 125px;
    height: 50px;
}

.title {
    font-family: 'Courier New', Courier, monospace;
    /* Change to the font family you need */
    font-size: 20px;
    /* Adjust size as needed */
    font-weight: bold;
    color: #333;
    /* Adjust color as needed */
    margin-left: 10px;
}

.main-container {
    height: 85vh;
    display: flex;
    background-color: #f2fbff;
    margin-left: 50px;
    margin-right: 50px;
    border-radius: 25px;
}

.aside-container {
    padding: 20px;
    display: flex;
    flex-direction: column;

    border-radius: 25px;
    margin-left: 10px;
    margin-top: 10px;
    margin-bottom: 10px;
}

.aside-header {
    display: flex;
    flex-direction: column;
    gap: 10px;
}

.aside-menu {
    margin-top: 20px;
}

.history-list {
    flex-grow: 1;
    margin-top: 20px;
    overflow-y: auto;
}

.history-item {
    padding: 10px;
    border-bottom: 1px solid #dcdcdc;
}



.feedback {
    font-family: 'Arial', sans-serif;
    font-size: 14px;
    color: #333;
}

.main-content {
    padding: 20px;
    overflow-y: auto;
    background-color: #e0e0e0;
    margin-right: 0px;
    background-image: url(../../imgs/bgsmall.png);
    background-size: cover;
    /* Ensure the image covers the entire container */
    background-position: center;
    /* Center the image */
    background-repeat: no-repeat;
    margin-bottom: 10px;
    height: 50vh;
    border-radius: 25px;
}

.footer-container {
    text-align: center;
    padding: 10px;
    background: rgba(255, 255, 255, 0.8);
    border-top: 1px solid #e0e0e0;
}

.menu-item-history:hover {
    border-radius: 10px;
    /* 圆角 */
}

.chat-container {
    /* max-height: 600px; */
    /* 根据需要设置 */
    overflow-y: auto;
    margin-left: 20px;
    margin-right: 20px;
    /* width: 100%; */
    /* overflow: hidden; */
}

/* 自定义滚动条样式 */
.chat-container::-webkit-scrollbar {
    width: 6px;
    /* 滚动条宽度 */
}

.chat-container::-webkit-scrollbar-track {
    background: transparent;
    /* 滚动条轨道颜色 */
    border-radius: 10px;
    /* 轨道圆角 */
}

.chat-container::-webkit-scrollbar-thumb {
    background: transparent;
    /* 滚动条拇指颜色 */
    border-radius: 10px;
    /* 拇指圆角 */
}

.chat-container::-webkit-scrollbar-thumb:hover {
    background: transparent;
    /* 鼠标悬停时的颜色 */
}

.chat-message {
    justify-content: space-between;
    margin-bottom: 20px;
    /* 调整消息之间的间距 */
}

.question-message {
    display: flex;
    align-items: center;
    justify-content: flex-end;
    /* 将提问消息放置在右边 */
    margin-right: 20px;
    /* 调整提问消息与边缘的距离 */
}

.answer-message {
    display: flex;
    align-items: center;
    justify-content: flex-start;
    /* 将回答消息放置在左边 */
    margin-left: 20px;
    overflow: hidden;
    border-radius: 15px;
    /* 调整回答消息与边缘的距离 */
}

.card {
    border-radius: 15px;
    padding: 10px;
    background-color: #1385f6;
    color: #fff;
    display: inline-block;
    font-size: 15px;
    word-wrap: break-word;
    padding: 10px;
    line-height: 1.2;
}

.input-wrapper {
    display: flex;
    align-items: center;
    background-color: #fff;
    border-radius: 25px;
    width: 100%;
    height: 60px;
    justify-content: space-between;
    margin-bottom: 10px
}

.input-button {
    display: flex;
    align-items: center;
    background-color: transparent;
    justify-content: space-around;
    flex-direction: row;
    margin-right: 10px;
}

.input-select1 {
    margin-left: 20px;
}


.reference-item {
    padding: 5px 0;
    /* Add padding between references */
}

.reference-content {
    display: block;
    white-space: nowrap;
    /* Prevent text from wrapping */
    overflow: hidden;
    text-overflow: ellipsis;
    /* Show ellipsis for overflow text */
    cursor: pointer;
}

.reference-content:hover {
    color: blue;
}

.custom-scrollbar::-webkit-scrollbar {
    width: 2px;
    /* Width of the vertical scrollbar */
}

.custom-scrollbar::-webkit-scrollbar-track {
    background: #f1f1f1;
    /* Color of the scrollbar track */
    border-radius: 10px;
    /* Round corners of the track */
}

.custom-scrollbar::-webkit-scrollbar-thumb {
    background: #888;
    /* Color of the scrollbar thumb */
    border-radius: 10px;
    /* Round corners of the thumb */
    border: 2px solid #f1f1f1;
    /* Space around the thumb */
}

.custom-scrollbar::-webkit-scrollbar-thumb:hover {
    background: #555;
    /* Color of the scrollbar thumb when hovered */
}

.custom-scrollbar {
    scrollbar-width: thin;
    /* For Firefox: make scrollbar thin */
    scrollbar-color: #888 #f1f1f1;
    /* For Firefox: thumb and track color */
}
:deep(.align-center) {
    font-weight: bolder;
}
</style>
<style scoped>
/* 添加新的样式 */
:deep(.preview-dialog .el-dialog__body) {
  padding: 0;
  display: flex;
  justify-content: center;
  align-items: center;
  max-height: 80vh;
}

.image-container {
  width: 100%;
  height: 100%;
  display: flex;
  justify-content: center;
  align-items: center;
  overflow: hidden;
}

.preview-image {
  max-width: 100%;
  max-height: 100%;
  object-fit: contain;
}


:deep(.el-dialog){
    display: flex;
    flex-direction: column;
    margin:0 !important;
    position:absolute;
    top:50%;
    left:50%;
    transform:translate(-50%,-50%);
    max-height:calc(100% - 30px);
    max-width:calc(100% - 30px);
}
:deep(.el-dialog .el-dialog__body){
    flex:1;
    overflow: auto;
}
</style>
