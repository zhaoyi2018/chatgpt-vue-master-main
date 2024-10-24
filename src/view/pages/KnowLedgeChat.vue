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
        
        <!-- 页面主体 -->
        <el-main style="padding: 0px;">
            <el-container class="main-container">
                <!-- 左侧边栏 -->
                <el-aside width="220px" class="aside-container">
                    <!-- 新对话按钮 -->
                    <el-header class="aside-header">
                        <el-button type="primary" icon="el-icon-plus" @click="newChat" round>新对话</el-button>
                    </el-header>
                    
                    <!-- 对话历史列表 -->
                    <el-menu @select="handleSelect" :default-active="activeIndex" class="custom-scrollbar"
                        style="background-color: #f2fbff; justify-content: center;  height:70vh; margin-bottom:20px; overflow-x: hidden; ">
                        <el-menu-item v-for="(question, index) in historyArrlist" :key="index" :index="index.toString()"
                            class="menu-item-history" @click="listConversion(index)">
                            <div slot="title" @mouseover="showDeleteButton(index)"
                                @mouseleave="hideDeleteButton(index)" class="menu-item-wrapper">
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
                        <div v-if="chatStarted" class="chat-container" ref="chatContainer">
                            <!-- 聊天消息循环 -->
                            <div v-for="(message, index) in chatMessages" :key="index" class="chat-message">
                                <!-- 用户消息 -->
                                <div v-if="message.role === 'user'" class="answer-message">
                                    <div class="card">
                                        <i class="el-icon-user" style="line-height: 1.5;"> {{ message.content }}</i>
                                    </div>
                                </div>
                                <!-- AI助手消息 -->
                                <div v-else-if="message.role === 'assistant'" class="answer-message">
                                    <div class="card" style=" background-color: #fff; float: left; color:#000; ">
                                        <!-- 参考资料显示 -->
                                        <div v-if="message.reference.length > 0">
                                            <i class="el-icon-paperclip" style="margin-top: 10px;margin-bottom: 10px;">Reference</i>
                                            <div v-for="(item, index1) in message.reference" :key="index1" class="reference-item">
                                                <div class="reference-content" @click="showFullContent(item)">
                                                    <span class="reference-number">{{ index1 + 1 }}. </span>
                                                    <el-tag v-if="item.standard_name" size="mini">{{ item.standard_name }}-{{ item.standard_clause }}-{{ item.standard_number }}</el-tag>
                                                    <el-tag v-else size="mini">{{ item.table }}</el-tag>
                                                    <span class="reference-text">
                                                        {{ truncateText(item.content, 20) }}
                                                    </span>
                                                </div>
                                            </div>
                                            <el-divider></el-divider>
                                        </div>
                                        <!-- 消息内容 -->
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
                            <div class="input-field-wrapper">
                                <el-input v-model="newMessage" class="input-field" placeholder="请输入内容"
                                    @input="sendMessage" @keyup.enter.native="startChat">
                                </el-input>
                            </div>
                            <div class="input-button">
                                <i class="el-icon-s-promotion" @click="startChat" style="margin-right: 5px;"></i>
                            </div>
                        </div>
                    </el-footer>
                </el-container>
            </el-container>
        </el-main>
        
        <!-- 相关内容弹窗组件 -->
        <el-dialog
            :visible.sync="dialogVisible"
            :title="fullReference"
            width="50%"
            align-center
        >
            <vue-markdown :source="fullContent" @rendered="addClickEvents"></vue-markdown>
        </el-dialog>
        
        <!-- 图片预览对话框 -->
        <el-dialog
            :visible.sync="previewDialogVisible"
            title="图片预览"
            width="50%"
            append-to-body
            align-center
        >
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
import commonMethodsMixin from '../../util/publicfun.js';
import VueMarkdown from 'vue-markdown';

export default {
    mixins: [commonMethodsMixin],
    components: {
        VueMarkdown
    },
    data() {
        return {
            // 默认知识库id
            defaultKbId: ['89bef038-8132-11ef-9800-2cf05d3470d1', '969c9f22-8132-11ef-8470-2cf05d3470d1', 'cae9d17d-8092-11ef-a840-2cf05d3470d1'],
            // 左侧激活对话索引(0开始)
            activeIndex: null,
            // 对话id
            dialogue_id: "",
            // 历史对话框列表
            historyArrlist: [],
            // 输入框内容（用户问话内容）
            newMessage: '',
            // 聊天消息（包括用户问话和回答）
            chatMessages: [],
            // 聊天是否开始
            chatStarted: false,
            // 弹窗组件
            dialogVisible: false,
            fullContent: '',
            fullReference: '',
            previewDialogVisible: false,
            previewUrl: "无图片",
            abortController: null,
        };
    },
    created() {
        // 获取token并初始化对话历史
        this.initializeChat();
    },
    watch: {
        chatMessages: {
            handler(newMessages) {
                this.scrollToBottom();
                this.initializeMessageReferences(newMessages);
            },
            immediate: true,
            deep: true
        }
    },
    methods: {
        // 初始化聊天
        async initializeChat() {
            const token = await this.getToken();
            await this.getChatHistory(token);
        },
        
        // 获取token
        async getToken() {
            const res = await login();
            return res.data.data.access_token;
        },
        
        // 获取对话历史列表
        async getChatHistory(token) {
            try {
                const res = await getChat({ access_token: token });
                this.dialogue_id = res.id;
                this.chatStarted = true;
                this.historyArrlist = res.data;
            } catch (err) {
                console.error("获取对话历史失败", err);
            }
        },
        
        // 初始化消息引用
        initializeMessageReferences(messages) {
            messages.forEach(message => {
                if (!message.reference) {
                    this.$set(message, 'reference', []);
                }
                if (!message.isHovered) {
                    this.$set(message, 'isHovered', Array(message.reference.length).fill(false));
                }
            });
        },
        
        // 截取内容
        truncateText(text, maxLength) {
            return text.length <= maxLength ? text : text.slice(0, maxLength) + '...';
        },
        
        // 显示完整内容
        showFullContent(item) {
            if (item.standard_name) {
                this.fullReference = `${item.standard_name}-${item.standard_clause}-${item.standard_number}`;
                this.fullContent = this.convertBracketsToSpans(item.content, item.image ? `http://39.106.94.192:8001/image/${item.image}` : "无图片");
            } else {
                this.fullReference = item.table;
                this.fullContent = this.appendCustomLink(item.content, item.table, item.image ? `http://39.106.94.192:8001/image/${item.image}` : "无图片");
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
        
        // 滚动到底部
        scrollToBottom() {
            this.$nextTick(() => {
                const container = this.$refs.chatContainer;
                if (container) {
                    container.scrollTop = container.scrollHeight;
                }
            });
        },
        
        // 显示删除按钮
        showDeleteButton(index) {
            this.$set(this.historyArrlist[index], 'showDeleteButton', true);
        },
        
        // 隐藏删除按钮
        hideDeleteButton(index) {
            this.$set(this.historyArrlist[index], 'showDeleteButton', false);
        },
        
        // 列出对话内容
        async listConversion(index) {
            try {
                const params = {
                    id: this.historyArrlist[index].id,
                    token: "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJuYW1lIjoidGVzdDEiLCJleHAiOjE3Mjg4Nzc1MDl9.10pwn0YnmSqIe7Ixsfozf1wDbk7RF4dn4KKc1NQWe7g"
                };
                const res = await list_conversion(params);
                this.dialogue_id = this.historyArrlist[index].id;
                this.chatStarted = true;
                this.chatMessages = [];
                res.data.data.forEach(item => {
                    this.chatMessages.push({ content: item.question, role: 'user' });
                    this.chatMessages.push({ content: item.answer, role: 'assistant', reference: item.reference });
                });
            } catch (err) {
                console.error("获取对话内容失败", err);
            }
        },
        
        // 删除对话项
        deleteItem(index) {
            const params = {
                dialog_id: this.historyArrlist[index].id,
                token: "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJuYW1lIjoidGVzdDEiLCJleHAiOjE3Mjg4Nzc1MDl9.10pwn0YnmSqIe7Ixsfozf1wDbk7RF4dn4KKc1NQWe7g",
            };
            this.$confirm('此操作将永久删除该对话, 是否继续?', '提示', {
                confirmButtonText: '确定',
                cancelButtonText: '取消',
                type: 'warning'
            }).then(() => {
                this.performDelete(params, index);
            }).catch(() => {
                this.$message({
                    type: 'info',
                    message: '已取消删除'
                });
            });
        },
        
        // 执行删除操作
        async performDelete(params, index) {
            try {
                const res = await delete_dialogue(params);
                if (res.data.code === 200) {
                    this.$message({
                        type: 'success',
                        message: '删除成功!'
                    });
                    this.historyArrlist.splice(index, 1);
                    this.newMessage = "";
                    this.chatStarted = false;
                    this.updateActiveIndex(index);
                } else {
                    this.$message({
                        type: 'error',
                        message: '删除失败!'
                    });
                }
            } catch (err) {
                console.error("删除对话失败", err);
            }
        },
        
        // 更新激活索引
        updateActiveIndex(deletedIndex) {
            this.$nextTick(() => {
                if (this.activeIndex != null) {
                    if (parseInt(this.activeIndex) > deletedIndex) {   
                        this.activeIndex = (parseInt(this.activeIndex) - 1).toString();
                    } else if (parseInt(this.activeIndex) == deletedIndex) {
                        this.activeIndex = null;
                        this.dialogue_id = "";
                    }
                }   
                document.activeElement.blur();
            });
        },
        
        // 处理选择
        handleSelect(index) {
            this.activeIndex = index.toString();
        },
        
        // 发送消息
        sendMessage(text) {
            this.newMessage = text;
        },
        
        // 开始聊天
        async startChat() {
            this.abortController = new AbortController();
            
            if (this.newMessage.trim() !== '' && this.newMessage.trim().length > 0) {
                if (this.dialogue_id === "") {
                    this.chatMessages = [];
                    await this.newChat();
                    await new Promise(resolve => setTimeout(resolve, 2000));
                }
                this.chatMessages.push({ content: this.newMessage, role: 'user' });
                const params = {
                    dialog_id: this.dialogue_id,
                    query: this.newMessage,
                    token: "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJuYW1lIjoidGVzdDEiLCJleHAiOjE3Mjg4Nzc1MDl9.10pwn0YnmSqIe7Ixsfozf1wDbk7RF4dn4KKc1NQWe7g",
                };
                chatkbStreamgpt(params, this.handleChunk, this.handleReferences, this.abortController.signal);
            } else {
                this.$alert('请输入内容', '提示', {
                    confirmButtonText: '确定',
                    callback: action => {}
                });
            }
        },
        
        // 处理聊天响应块
        handleChunk(first, content, reference, end) {
            if (first) {
                if (this.chatMessages.length == 1) {  
                    this.updateDialogName();
                }
                this.chatMessages.push({ content: '', role: 'assistant', reference: JSON.parse(reference).reference });
                this.newMessage = '';
            }
            const lastMessageIndex = this.chatMessages.length - 1;
            if (!end) {
                this.chatMessages[lastMessageIndex].content += content;
                this.$set(this.chatMessages, lastMessageIndex, { ...this.chatMessages[lastMessageIndex] });
            } else {
                // this.chatMessages[lastMessageIndex].content = JSON.parse(content).response;
                // this.$set(this.chatMessages, lastMessageIndex, { ...this.chatMessages[lastMessageIndex] });
                this.chatStarted = true;
            } 
        },
        
        // 更新对话名称
        updateDialogName() {
            this.historyArrlist[parseInt(this.activeIndex)].name = this.newMessage;
            const params = {
                ...this.historyArrlist[parseInt(this.activeIndex)],
                token: "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJuYW1lIjoidGVzdDEiLCJleHAiOjE3Mjg4Nzc1MDl9.10pwn0YnmSqIe7Ixsfozf1wDbk7RF4dn4KKc1NQWe7g",
                dialog_id: this.historyArrlist[parseInt(this.activeIndex)].id,
                name: this.newMessage,
                update_time: new Date().toISOString()
            };
            updateDialog(params).then((res) => {
                console.log("对话名称更新成功", res);
            }).catch(err => {
                console.error("对话名称更新失败", err);
            });
        },
        
        // 处理参考内容
        handleReferences(reference) {
            const parsedReference = JSON.parse(reference);
            const lastMessageIndex = this.chatMessages.length - 1;
            this.chatMessages[lastMessageIndex].content = parsedReference.response;
            this.chatMessages[lastMessageIndex].reference = parsedReference.reference;
            this.$set(this.chatMessages, lastMessageIndex, { ...this.chatMessages[lastMessageIndex] });
        },
        
        // 新建聊天
        async newChat() {
            if (this.chatMessages.length != 0) {
                this.chatMessages = [];
            }
            const params = {
                token: "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJuYW1lIjoidGVzdDEiLCJleHAiOjE3Mjg4Nzc1MDl9.10pwn0YnmSqIe7Ixsfozf1wDbk7RF4dn4KKc1NQWe7g",
                kb_ids: this.defaultKbId,
                dialog_type: 1,
                name: "对话1"
            };
            try {
                const res = await getkbChat(params);
                this.dialogue_id = res.data.dialog;
                this.chatStarted = true;
                const newhistory = {
                    id: this.dialogue_id,
                    name: '新对话',
                    dialog_type: 1,
                    update_time: new Date().toISOString(),
                    kb_ids: this.defaultKbId,
                };
                this.historyArrlist.unshift(newhistory);
                this.activeIndex = '0';
            } catch (err) {
                console.error("创建新对话失败", err);
            }
        },
        
        // 添加新方法
        goToHome() {
            this.$router.push('/ChatHome');
        }
    },
    beforeRouteLeave(to, from, next) {
        if (this.abortController) {
            this.abortController.abort();
        }
        next();
    },
    beforeDestroy() {
        if (this.abortController) {
            this.abortController.abort();
        }
    },
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
    justify-content: space-between;
    align-items: center;
    padding: 0 20px;
    height: 60px;
}

.header-left {
    display: flex;
    align-items: center;
    margin-left: 27px;
}

.header-right {
    display: flex;
    align-items: center;
    justify-content: center; /* 添加这行来使内容居中 */
    flex-grow: 1; /* 添加这行使header-right占据剩余空间 */
}

.el-button--text {
    color: #1385f6;
    font-size: 16px;
}

.el-button--text:hover {
    color: #409EFF;
}

.logo {
    max-height: 40px; /* 调整logo的最大高度 */
    width: auto;
    object-fit: contain;
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
    height: calc(100vh - 200px); /* 调整这个值以适应你的布局 */
    overflow-y: hidden; /* 防止双重滚动条 */
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
    height: 100%;
    overflow-y: auto;
    padding-right: 15px; /* 为滚动条留出空间 */
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
    background: #f1f1f1;
    /* 滚动条轨道颜色 */
    border-radius: 10px;
    /* 轨道圆角 */
}

.chat-container::-webkit-scrollbar-thumb {
    background: #888;
    /* 滚动条拇指颜色 */
    border-radius: 10px;
    /* 拇指圆角 */
}

.chat-container::-webkit-scrollbar-thumb:hover {
    background: #555;
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






