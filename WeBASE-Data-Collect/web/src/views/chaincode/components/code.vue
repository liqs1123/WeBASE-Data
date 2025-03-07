/*
 * Copyright 2014-2020 the original author or authors.
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *      http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
 
<template>
    <div class="contract-code" :class="{changeActive:changeWidth }" v-loading="loading">
        <div class="contract-code-head">
            <span class="contract-code-title" v-show="codeShow" :class="{titleActive:changeWidth }">
                <span>{{contractName + '.sol'}}</span>
            </span>
            <span class="contract-code-handle" v-show="codeShow">
                <span class="contract-code-done" v-if="!contractAddress" @click="saveCode">
                    <el-tooltip class="item" effect="dark" content="按Ctrl+s保存合约内容" placement="top-start">
                        <i class="wbs-icon-baocun font-16"></i>
                    </el-tooltip>
                    <span>保存</span>
                </span>
                <span class="contract-code-done" @click="compile" v-if="!contractAddress">
                    <i class="wbs-icon-bianyi font-16"></i>
                    <span>编译</span>
                </span>
            </span>
        </div>
        <div class="contract-code-content" :class="{infoHide: !successHide}">
            <div class="contract-code-mirror" :style="{height:codeHight}" ref="codeContent">
                <div style="padding-top: 60px;text-align:center;" v-show="!codeShow">
                    <span>请在左侧面板点击打开一个合约或新建一个合约</span>
                </div>
                <div class="ace-editor" ref="ace" v-show="codeShow"></div>
            </div>
            <div class="contract-info" v-show="successHide" :style="{height:infoHeight + 'px'}">
                <div class="move" @mousedown="dragDetailWeight($event)" @mouseup="resizeCode"></div>
                <div class="contract-info-title" @mouseover="mouseHover=!mouseHover" @mouseout="mouseHover=!mouseHover" @click="collapse" v-show="abiFile||contractAddress">
                    <i :class="[showCompileText?'el-icon-caret-bottom':'el-icon-caret-top']">

                    </i>
                </div>
                <div>
                    <div class="contract-info-list1" v-html="compileinfo">
                    </div>
                    <div class="contract-info-list1" style="color: #f00" v-show="errorInfo">
                        {{errorInfo}}
                    </div>
                    <div class="contract-info-list1" style="color: #f00" v-show="errorInfo">
                        <span style="display:inline-block;width:calc(100% - 120px);word-wrap:break-word" v-for="(item, index) in errorMessage">{{index+1}}、{{item}}</span>
                    </div>
                    <div style="color: #68E600;padding-bottom: 15px;" v-show="abiFileShow">{{successInfo}}</div>
                    <div class="contract-info-list" v-show="contractAddress">
                        <span class="contract-info-list-title" style="color: #0B8AEE">contractAddress
                            <i class="wbs-icon-copy font-12 copy-public-key" @click="copyKey(contractAddress)" title="复制"></i>
                        </span>
                        <span style="display:inline-block;width:calc(100% - 120px);word-wrap:break-word">{{contractAddress}}</span>
                    </div>
                    <div class="contract-info-list" v-show="abiFile">
                        <span class="contract-info-list-title" style="color: #0B8AEE">contractName
                            <i class="wbs-icon-copy font-12 copy-public-key" @click="copyKey(contractName)" title="复制"></i> </span>
                        <span style="display:inline-block;width:calc(100% - 120px);word-wrap:break-word">{{contractName}}</span>
                    </div>
                    <div class="contract-info-list" v-show="abiFile">
                        <span class="contract-info-list-title" style="color: #0B8AEE">abi
                            <i class="wbs-icon-copy font-12 copy-public-key" @click="copyKey(abiFile)" title="复制"></i>
                        </span>
                        <span class="showText" ref="showAbiText">
                            {{abiFile}}
                        </span>
                        <i :class="[ showAbi ? 'el-icon-arrow-down': 'el-icon-arrow-up','font-13','cursor-pointer','visibility-wrapper'] " v-if="complieAbiTextHeight" @click="showAbiText"></i>
                    </div>
                    <div class="contract-info-list" style="border-bottom: 1px solid #e8e8e8" v-show="abiFile">
                        <span class="contract-info-list-title" style="color: #0B8AEE">bytecodeBin
                            <i class="wbs-icon-copy font-12 copy-public-key" @click="copyKey(bytecodeBin)" title="复制"></i>
                        </span>
                        <span class="showText" ref="showBinText">
                            {{bytecodeBin}}
                        </span>
                        <i :class="[ showBin ? 'el-icon-arrow-down': 'el-icon-arrow-up','font-13','cursor-pointer','visibility-wrapper'] " v-if="complieBinTextHeight" @click="showBinText"></i>
                    </div>
                </div>
            </div>
        </div>
        <v-editor v-if='editorShow' :show='editorShow' :data='editorData' :input='editorInput' :editorOutput="editorOutput" @close='editorClose'></v-editor>
        <v-upload v-if='uploadFileAdrShow' :show='uploadFileAdrShow' @close='uploadClose' @success='uploadSuccess($event)'></v-upload>
    </div>
</template>

<script>
import ace from "ace-builds";
// import "ace-builds/webpack-resolver";
import "ace-builds/src-noconflict/theme-chrome";
import "ace-builds/src-noconflict/mode-javascript";
import "ace-builds/src-noconflict/ext-language_tools";
require("ace-mode-solidity/build/remix-ide/mode-solidity");
let Mode = require("ace-mode-solidity/build/remix-ide/mode-solidity").Mode;
import errcode from "@/util/errcode";
let Base64 = require("js-base64").Base64;
import constant from "@/util/constant";
import editor from "../dialog/editor"
import uploadFileAdr from "../dialog/uploadFileAdr"
import Bus from '@/bus'
import web3 from "@/util/ethAbi"
import router from "@/router"
import {
    addChaincode,
    getDeployStatus,
    setCompile,
    editChain,
    addFunctionAbi,
    backgroundCompile
} from "@/util/api";

export default {
    
    name: "codes",
    props: ["show", "changeStyle"],
    components: {
        "v-editor": editor,
        "v-upload": uploadFileAdr
    },
    data() {
        return {
            successHide: true,
            loading: false,
            content: "",
            successShow: true,
            errorShow: false,
            dialogVisible: false,
            code: "",
            status: 0,
            abiFile: "",
            bin: "",
            contractAddress: "",
            contractName: "",
            infoHeight: 250,
            contractList: [],
            compileinfo: "",
            errorInfo: "",
            errorMessage: "",
            successInfo: "",
            abiFileShow: false,
            bytecodeBin: "",
            aceEditor: null,
            themePath: "ace/theme/chrome",
            modePath: "ace/mode/solidity",
            data: null,
            codeShow: false,
            version: "",
            saveShow: false,
            editorShow: false,
            editorData: null,
            editorInput: null,
            editorOutput: null,
            uploadFileAdrShow: false,
            uploadAddress: "",
            showAbi: true,
            showBin: true,
            complieAbiTextHeight: false,
            complieBinTextHeight: false,
            mouseHover: false,
            showCompileText: true
        };
    },
    beforeDestroy() {
        Bus.$off("select")
        Bus.$off("noData")
    },
    beforeMount() {

    },
    mounted() {
        this.initEditor();
        Bus.$on('select', data => {
            this.codeShow = true;
            this.refreshMessage();
            this.code = "";
            this.version = "";
            this.status = null;
            this.abiFile = "";
            this.contractAddress = "";
            this.errorMessage = "";
            this.contractName = "";
            this.content = "";
            this.bin = "";
            this.data = data;
            this.code = Base64.decode(data.contractSource);
            this.content = this.code;
            this.aceEditor.setValue(this.content);
            this.status = data.contractStatus;
            this.abiFile = data.contractAbi;
            this.contractAddress = data.contractAddress;
            this.errorMessage = data.description || "";
            this.contractName = data.contractName;
            this.bin = data.runtimeBin;
            this.bytecodeBin = data.bytecodeBin || "";
            this.version = data.contractVersion;
            this.complieAbiTextHeight = false;
            this.complieBinTextHeight = false;
        })
        Bus.$on("noData", data => {
            this.codeShow = false;
            this.refreshMessage();
            this.code = "";
            this.version = "";
            this.status = null;
            this.abiFile = "";
            this.contractAddress = "";
            this.errorMessage = "";
            this.contractName = "";
            this.content = "";
            this.bin = "";
        })
    },
    watch: {
        content(val) {
            let data = Base64.decode(this.data.contractSource);
            if (data != val) {
                this.saveShow = true
            } else {
                this.saveShow = false
            }
        },
        successHide(val) {
            if (val) {
                this.infoHeight = 250;
            } else {
                this.infoHeight = 0;
            }
        }
    },
    computed: {
        codeHight() {
            if (this.infoHeight) {
                return `calc(100% - ${this.infoHeight}px)`;
            } else {
                return `100%`;
            }
        },
        changeWidth() {
            if (this.changeStyle) {
                return this.changeStyle;
            } else {
                return false;
            }
        },
        tipShow() {
            return !this.show;
        }
    },
    methods: {
        initEditor() {
            let _this = this
            this.aceEditor = ace.edit(this.$refs.ace, {
                fontSize: 14,
                fontFamily: "Consolas,Monaco,monospace",

                theme: this.themePath,
                mode: this.modePath,
                tabSize: 4,
                useSoftTabs: true
            });
            this.aceEditor.setOptions({
                enableSnippets: true,
                enableLiveAutocompletion: true,
                enableBasicAutocompletion: true,
                autoScrollEditorIntoView: true,
                copyWithEmptySelection: true
            });
            this.aceEditor.commands.addCommand({
                name: 'myCommand',
                bindKey: { win: 'Ctrl-S', mac: 'Command-S' },
                exec(editor) {
                    if (_this.data.contractStatus != 2) {
                        _this.saveCode()
                    }
                },
            });
            let editor = this.aceEditor.alignCursors();
            this.aceEditor.getSession().setUseWrapMode(true);
            this.aceEditor.getSession().on("change", this.changeAce);
            this.aceEditor.on("blur", this.blurAce);
            this.aceEditor.resize();
            this.aceEditor.setReadOnly(true);
        },
        blurAce() {
            let data = Base64.encode(this.content);
            if (this.data.contractSource != data && this.data.contractStatus != 2) {
                this.saveCode()
            }
        },
        saveCode() {
            this.data.contractSource = Base64.encode(this.content);
            Bus.$emit("save", this.data)
        },
        resizeCode() {
            this.aceEditor.setOptions({
                maxLines:
                    Math.ceil(this.$refs.codeContent.offsetHeight / 17) - 1
            });
            this.aceEditor.resize();
        },
        dragDetailWeight(e) {
            let startY = e.clientY,
                infoHeight = this.infoHeight;
            document.onmousemove = e => {
                let moveY = startY - e.clientY;
                this.infoHeight = infoHeight + moveY;
            };
            document.onmouseup = e => {
                document.onmousemove = null;
                document.onmouseup = null;
            };
            this.aceEditor.setOptions({
                maxLines:
                    Math.ceil(this.$refs.codeContent.offsetHeight / 17) - 1,
                minLines: 9
            });
        },
        upLoadAdr() {
            this.uploadFileAdrShow = true
        },
        uploadClose() {
            this.uploadFileAdrShow = false
        },
        uploadSuccess(val) {
            this.dialogVisible = true;
            this.uploadAddress = val
        },
        editorClose() {
            this.editorShow = false;
        },
        changeAce() {
            this.content = this.aceEditor.getSession().getValue();
        },

        findImports(path) {
            this.contractList = JSON.parse(
                localStorage.getItem("contractList")
            );
            let arry = path.split("/");
            let newpath = arry[arry.length - 1];
            let num = 0;
            if (arry.length > 1) {
                let newPath = arry[0]
                let oldPath = arry[arry.length - 1]
                let importArry = []
                this.contractList.forEach(value => {
                    if (value.contractPath == newPath) {
                        importArry.push(value)
                    }
                })
                if (importArry.length) {
                    for (let i = 0; i < importArry.length; i++) {
                        if (oldPath == importArry[i].contractName + ".sol") {
                            return {
                                contents: Base64.decode(
                                    importArry[i].contractSource
                                )
                            };
                        }
                    }
                } else {
                    return { error: "File not found" };
                }
            } else {
                let newpath = arry[arry.length - 1];
                let newArry = []
                this.contractList.forEach(value => {
                    if (value.contractPath == this.data.contractPath) {
                        newArry.push(value)
                    }
                })
                if (newArry.length > 1) {
                    for (let i = 0; i < newArry.length; i++) {
                        if (newpath == newArry[i].contractName + ".sol") {
                            return {
                                contents: Base64.decode(
                                    newArry[i].contractSource
                                )
                            };
                        }
                    }
                    for (let i = 0; i < this.contractList.length; i++) {
                        if (newpath == this.contractList[i].contractName + ".sol") {
                            return {
                                contents: Base64.decode(this.contractList[i].contractSource)
                            };
                        } else {
                            num++;
                        }
                    }
                    if (num) {
                        return { error: "File not found" };
                    }
                } else {
                    for (let i = 0; i < this.contractList.length; i++) {
                        if (newpath == this.contractList[i].contractName + ".sol") {
                            return {
                                contents: Base64.decode(this.contractList[i].contractSource)
                            };
                        }
                    }
                }
            }
        },
        compile() {
            let wrapper = require("solc/wrapper");
            let solc = wrapper(window.Module);
            this.loading = true;
            this.refreshMessage();
            for (let i = 0; i < constant.COMPILE_INFO.length; i++) {
                this.compileinfo = this.compileinfo + constant.COMPILE_INFO[i];
            }
            this.contractList = JSON.parse(
                localStorage.getItem("contractList")
            );
            let content = "";
            let output;
            let input = {
                language: "Solidity",
                settings: {
                    outputSelection: {
                        "*": {
                            "*": ["*"]
                        }
                    }
                }
            };
            input.sources = {};
            input.sources[this.contractName + ".sol"] = {};
            let libs = [];
            input.sources[this.contractName + ".sol"] = {
                content: this.content
            };
            try {
                output = JSON.parse(solc.compileStandard(JSON.stringify(input), this.findImports));
            } catch (error) {
                this.errorInfo = "合约编译失败！";
                this.errorMessage = error;
                this.compileShow = true;
                this.loading = false;
            }
            setTimeout(() => {
                if (output && JSON.stringify(output.contracts) != "{}") {
                    this.status = 1;
                    if (output.contracts[this.contractName + ".sol"]) {
                        this.changeOutput(
                            output.contracts[this.contractName + ".sol"]
                        );
                    }
                } else {
                    console.log('output:',output)
                    if(output)this.errorMessage = output.errors;
                    this.errorInfo = "合约编译失败！";
                    this.loading = false;
                }
            }, 500)
        },
        changeOutput(obj) {
            if (JSON.stringify(obj) !== "{}") {
                if (obj.hasOwnProperty(this.contractName)) {
                    let compiledMap = obj[this.contractName]
                    this.abiFileShow = true;
                    this.successInfo = "< 编译成功！";
                    this.abiFile = compiledMap.abi;
                    this.abiFile = JSON.stringify(this.abiFile);
                    this.bin = compiledMap.evm.deployedBytecode.object;
                    this.bytecodeBin = compiledMap.evm.bytecode.object;
                    this.data.contractAbi = this.abiFile;
                    this.data.runtimeBin = this.bin;
                    this.data.contractSource = Base64.encode(this.content);
                    this.$set(this.data, "bytecodeBin", this.bytecodeBin);
                    this.loading = false;
                    Bus.$emit("compile", this.data);
                    this.setMethod();
                } else {
                    this.$message({
                        type: "error",
                        message: "合约名和文件名要保持一致!"
                    })
                    this.errorInfo = "合约编译失败！";
                    this.compileShow = true;
                    this.loading = false;
                }
            } else {
                this.errorInfo = "合约编译失败！";
                this.compileShow = true;
                this.loading = false;
            }
        },
        refreshMessage() {
            this.abiFileShow = false;
            this.errorInfo = "";
            this.compileinfo = "";
            this.abiFile = "";
            this.contractAddress = "";
        },
        setMethod() {
            let Web3EthAbi = web3;
            let arry = [];
            if (this.abiFile) {
                let list = JSON.parse(this.abiFile);
                list.forEach(value => {
                    if (value.name && value.type == 'function') {
                        let data = {}
                        let methodId;
                        if (localStorage.getItem("encryptType") == 1) {
                            methodId = Web3EthAbi.smEncodeFunctionSignature({
                                name: value.name,
                                type: value.type,
                                inputs: value.inputs
                            });
                        } else {
                            methodId = Web3EthAbi.encodeFunctionSignature({
                                name: value.name,
                                type: value.type,
                                inputs: value.inputs
                            });
                        }
                        data.methodId = methodId;
                        data.methodName = value.name;
                        data.methodType = value.type
                        arry.push(data)
                    } else if (value.name && value.type == 'event') {
                        let data = {}
                        let methodId;
                        if (localStorage.getItem("encryptType") == 1) {
                            methodId = Web3EthAbi.smEncodeEventSignature({
                                name: value.name,
                                type: value.type,
                                inputs: value.inputs
                            });
                        } else {
                            methodId = Web3EthAbi.encodeEventSignature({
                                name: value.name,
                                type: value.type,
                                inputs: value.inputs
                            });
                        }
                        data.methodId = methodId;
                        data.methodName = value.name;
                        data.methodType = value.type
                        arry.push(data)
                    }
                })
                if (arry.length) {
                    this.addAbiMethod(arry)
                }
            }
        },
        addAbiMethod(list) {
            let data = {
                contractId: this.data.contractId,
                methodList: list
            }
            addFunctionAbi(data).then(res => {
                if (res.data.code === 0) {
                    console.log("method 保存成功！")
                } else {
                    this.$message({
                        message: this.$chooseLang(res.data.code),
                        type: "error",
                        duration: 2000
                    });
                }
            }).catch(err => {
                this.$message({
                    message: "系统错误",
                    type: "error",
                    duration: 2000
                });
            })
        },
        foldInfo(val) {
            this.successHide = val;
        },
        showAbiText() {
            this.showAbi = !this.showAbi
            if (this.$refs['showAbiText'].style.overflow === 'visible') {
                this.$refs['showAbiText'].style.overflow = 'hidden'
            } else if (this.$refs['showAbiText'].style.overflow === '' || this.$refs['showAbiText'].style.overflow === 'hidden') {
                this.$refs['showAbiText'].style.overflow = 'visible'
            }
        },
        showBinText() {
            this.showBin = !this.showBin
            if (this.$refs['showBinText'].style.overflow === 'visible') {
                this.$refs['showBinText'].style.overflow = 'hidden'
            } else if (this.$refs['showBinText'].style.overflow === '' || this.$refs['showBinText'].style.overflow === 'hidden') {
                this.$refs['showBinText'].style.overflow = 'visible'
            }
        },
        showHover() {

        },
        hiddenHover() {
            
        },
        collapse() {
            this.showCompileText = !this.showCompileText
            if (this.showCompileText) {
                this.infoHeight = 250

            } else if (!this.showCompileText) {
                this.infoHeight = 50
            }
            this.$nextTick(() => {
                this.resizeCode()
            })

        },
        copyKey(val) {
            if (!val) {
                this.$message({
                    type: "fail",
                    showClose: true,
                    message: "key为空，不复制。",
                    duration: 2000
                });
            } else {
                this.$copyText(val).then(e => {
                    this.$message({
                        type: "success",
                        showClose: true,
                        message: "复制成功",
                        duration: 2000
                    });
                });
            }
        }

    }
};
</script>
<style scoped>
.contract-code {
    height: 100%;
    border-left: 1px solid #eee;
    box-sizing: border-box;
}
.changeActive {
    padding-left: 0 !important;
}
.contract-code-head {
    width: 100%;
    height: 48px;
    line-height: 48px;
    border-bottom: 2px solid #e7ebf0;
    background-color: #fff;
    overflow: hidden;
}
.contract-code-done {
    display: inline-block;
    margin-right: 10px;
    cursor: pointer;
}
.contract-code-done i {
    vertical-align: middle;
}
.contract-code-done span {
    font-size: 12px;
    color: #9da2ab;
    vertical-align: middle;
}
.contract-no-content {
    border-left: 1px solid #e7ebf0;
    height: calc(100% - 50px);
    box-sizing: border-box;
}
.contract-code-content {
    border-left: 1px solid #e7ebf0;
    height: calc(100% - 50px);
    box-sizing: border-box;
}
.contract-code-mirror {
    width: 100%;
    height: 70%;
}
.contract-info {
    position: relative;
    padding-top: 20px;
    text-align: left;
    border-top: 1px solid #ddd;
    box-sizing: border-box;
    overflow: auto;
}
.contract-info-content {
    height: 100%;
    overflow-y: scroll;
    box-sizing: border-box;
}
.contract-code-title {
    float: left;
    font-weight: bold;
    font-size: 18px;
    color: #36393d;
    padding-left: 20px;
}
.contract-code-handle {
    float: right;
    padding-right: 20px;
}
.contract-info-title {
    text-align: center;
    cursor: pointer;
}
.move {
    position: absolute;
    width: 100%;
    height: 3px;
    top: 0;
    left: 0;
    z-index: 9999;
    cursor: s-resize;
}
.contract-info-title i {
    padding-left: 8px;
    font-size: 10px;
    color: #aeb1b5;
    cursor: pointer;
}
.contract-info-title span {
    font-size: 16px;
    font-weight: bold;
    color: #36393d;
}
.contract-info-list {
    padding: 5px 20px;
    width: 90%;
    margin: 0 auto;
    border: 1px solid #e8e8e8;
    border-bottom: none;
    position: relative;
}
.contract-info-list-title {
    display: inline-block;
    width: 105px;
    vertical-align: top;
}
.contract-info-list-title::after {
    display: block;
    content: "";
    clear: both;
}
.ace-editor {
    height: 100% !important;
    position: relative;
    text-align: left;
    letter-spacing: 0.1px;
    text-rendering: geometricPrecision;
    font-feature-settings: "liga" 0;
    font-variant-ligatures: none;
    font: 14px / normal "Monaco", "Menlo", "Ubuntu Mono", "Consolas",
        "source-code-pro", monospace !important;
}
.ace-editor >>> .ace_print-margin {
    display: none;
    text-rendering: geometricPrecision;
}
.infoHide {
    height: calc(100% - 50px);
}
.code-spread {
    position: absolute;
    width: 33px;
    height: 33px;
    line-height: 33px;
    left: 412px;
    bottom: 0;
    border: 1px solid #e8e8e8;
    color: #aeb1b5;
    background-color: #fff;
    text-align: center;
    z-index: 9999;
    cursor: pointer;
}
.code-spread i {
    font-size: 12px;
}
.contract-info {
    background-color: #fff;
}
.titleActive {
    padding-left: 40px;
}
.send-dialog /deep/ .el-dialog--center .el-dialog__body {
    padding: 5px 25px 20px;
}
.showText {
    display: inline-block;
    width: calc(100% - 120px);
    word-wrap: break-word;
    max-height: 73px;
    overflow: hidden;
}
.visibility-wrapper {
    position: absolute;
    bottom: 10px;
}
.copy-public-key {
    float: right;
}
</style>
