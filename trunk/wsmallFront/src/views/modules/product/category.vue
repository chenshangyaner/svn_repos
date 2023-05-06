<template>
    <div>
        <el-switch v-model="draggable" active-text="编辑" inactive-text="">
        </el-switch>
        <el-button v-if="draggable" @click="batchSave" plain size="mini" style="float:right">批量保存</el-button>
        <el-button v-if="draggable" type="danger" @click="batchDelete" round size="mini">批量删除</el-button>
        <el-tree :data="menus" :props="defaultProps" node-key="catId" :show-checkbox="draggable"
            :expand-on-click-node="false" :default-expanded-keys="expandedKey" :draggable="draggable"
            :allow-drop="allowDrop" @node-drop="handleDrop" ref="menuTree">
            <span class="custom-tree-node" slot-scope="{ node, data }">
                <span>{{ node.label }}</span>
                <span>
                    <el-button v-if="node.level <= 2" type="text" size="mini" @click="() => append(data)">
                        添加
                    </el-button>
                    <el-button type="text" size="mini" @click="() => edit(data)">
                        修改
                    </el-button>
                    <el-button v-if="node.childNodes.length == 0" type="text" size="mini" @click="() => remove(node, data)">
                        删除
                    </el-button>
                </span>
            </span>
        </el-tree>
        <el-dialog :title="title" :visible.sync="dialogVisible" width="30%" :close-on-click-modal="false">
            <el-form :model="category">
                <el-form-item label="分类名称">
                    <el-input v-model="category.name" autocomplete="off"></el-input>
                </el-form-item>
                <el-form-item label="图标">
                    <el-input v-model="category.icon" autocomplete="off"></el-input>
                </el-form-item>
                <el-form-item label="计量单位">
                    <el-input v-model="category.productUnit" autocomplete="off"></el-input>
                </el-form-item>
            </el-form>
            <span slot="footer" class="dialog-footer">
                <el-button type="primary" @click="cancalForThis">取 消</el-button>
                <el-button type="primary" @click="submitData">确 定</el-button>
            </span>
        </el-dialog>
    </div>
</template>
  
<script>

export default {
    components: {},
    props: {},
    data() {
        return {
            pCid: [],
            draggable: false,
            updateNodes: [],
            result: false,
            maxLevel: 0,
            title: "",
            // 对话框类型
            dialogType: "", //edit,add
            category: { name: "", parentCid: 0, catLevel: 0, showStatus: 1, sort: 0, catId: null, icon: null, productUnit: null },
            dialogVisible: false,
            menus: [],
            expandedKey: [],
            defaultProps: {
                children: "children",
                label: "name"
            }

        };
    },


    methods: {

        // 获取menu数据
        getMenus() {
            this.$http({
                url: this.$http.adornUrl('/product/category/list/tree'),//后端接口
                method: 'get'
                //   // params表示携带的参数，可省略
                //   params: this.$http.adornParams({
                //     'page': this.pageIndex,
                //     'limit': this.pageSize,
                //     'roleName': this.dataForm.roleName
                //   })
            }).then(({ data }) => {
                console.log("成功获取", data.data);
                this.menus = data.data;
            });
        },

        // 添加节点内容
        append(data) {
            console.log("append", data)
            this.title = "添加";
            this.dialogType = "add";
            this.dialogVisible = true;
            //强制刷空？
            //this.category.name = "";

            this.category.parentCid = data.catId;
            this.category.catLevel = data.catLevel * 1 + 1;
        },
        //添加三级分类的内容
        addCategory() {
            console.log("添加的三级表单", this.category);
            this.$http({
                url: this.$http.adornUrl('/product/category/save'),
                method: 'post',
                data: this.$http.adornData(this.category, false)
            }).then(({ data }) => {
                this.$message({
                    message: "添加成功",
                    type: "success"
                });
                this.dialogVisible = false;
                this.getMenus();
                this.expandedKey = [this.category.parentCid]
            })
        },

        //修改 
        edit(data) {
            this.title = "修改";
            console.log(data);
            this.dialogType = "edit";
            this.dialogVisible = true;
            //回显 需要发送请求拿到最新的数据
            this.$http({
                url: this.$http.adornUrl(`/product/category/info/${data.catId}`),
                method: 'get',
            }).then(({ data }) => {
                this.category.name = data.data.name;
                this.category.catId = data.data.catId;
                this.category.icon = data.data.icon;
                this.category.productUnit = data.data.productUnit;
                this.category.parentCid = data.data.parentCid
            })

        },

        editCategory() {
            var { name, catId, icon, productUnit } = this.category;
            var data = { name: name, catId: catId, icon: icon, productUnit: productUnit };
            this.$http({
                url: this.$http.adornUrl('/product/category/update'),
                method: 'post',
                data: this.$http.adornData(data, false)
            }).then(({ data }) => {
                //success tips
                this.$message({
                    message: "修改成功",
                    type: "success"
                });
                // 刷新菜单
                this.getMenus();
                //删除完维持展开
                this.expandedKey = [this.category.parentCid];

            })
            this.dialogVisible = false;
        },

        // remove删除节点内容
        remove(node, data) {
            var ids = [data.catId];
            this.$confirm(`是否删除该文件【${data.name}】?`, '提示', {
                confirmButtonText: '确定',
                cancelButtonText: '取消',
                type: 'warning'
            }).then(() => {
                this.$http({
                    url: this.$http.adornUrl('/product/category/delete'),
                    method: 'post',
                    data: this.$http.adornData(ids, false)
                }).then(({ data }) => {
                    //success tips
                    this.$message({
                        message: "删除成功",
                        type: "success"
                    });
                    // 刷新菜单
                    this.getMenus();
                    //删除完维持展开
                    this.expandedKey = [node.parent.data.catId];
                })
            }).catch(() => {
                this.$message({
                    type: 'info',
                    message: '已取消删除'
                });
            });
        },

        cancalForThis() {
            this.dialogVisible = false;
            this.category.name = "";
            this.category.icon = "";
            this.category.productUnit = "";
        },
        submitData() {
            if (this.dialogType == "add") {
                this.addCategory();
            }
            if (this.dialogType == "edit") {
                this.editCategory();
            }
        },

        // 拖拽判断 根据自己的业务需求决定是否可以拖拽
        allowDrop(draggableNode, dropNode, type) {
            // 当前拖动节点的总层数
            this.countNodeLevel(draggableNode);

            // 当前正在拖动的节点+父节点的深度不大于3即可
            let deep = Math.abs((this.maxLevel - draggableNode.level)) + 1;
            if (type == "inner") {
                return this.result = (deep + dropNode.level) <= 3 ? true : false;
            } else {
                return this.result = (deep + dropNode.parent.level) <= 3 ? true : false;
            }
        },
        countNodeLevel(node) {
            //递归查找
            if (node.childNodes != null && node.childNodes.length > 0) {
                for (let i = 0; i < node.childNodes.length; i++) {
                    if (node.childNodes[i].level > this.maxLevel) {
                        this.maxLevel = node.childNodes[i].level;
                    }
                    this.countNodeLevel(node.childNodes[i]);
                }
            }

        },
        // 拖拽事件监听并执行
        handleDrop(draggingNode, dropNode, dropType, event) {
            //获取新的父节点id
            let pCid = 0;
            let siblings = null;
            if (dropType == "inner") {
                pCid = dropNode.data.catId;
                siblings = dropNode.childNodes;
            } else {
                pCid = dropNode.parent.data.catId == undefined ? 0 : dropNode.parent.data.catId;
                siblings = dropNode.parent.childNodes;
            }
            this.pCid.push(pCid);

            //新节点的排序 
            for (let i = 0; i < siblings.length; i++) {
                // 封装成对象，将其传递给数据库
                if (siblings[i].data.catId == draggingNode.data.catId) {
                    // 遍历到正在拖拽的节点，父ID要变化
                    let catLevel = draggingNode.data.catLevel;
                    if (siblings[i].level != draggingNode.level) {
                        // catLevel的更新值
                        catLevel = siblings[i].level;
                        // 修改子节点的层级 递归
                        this.updateChildrenNodeLevel(siblings[i])

                    }
                    this.updateNodes.push({ catId: siblings[i].data.catId, sort: i, parentCid: pCid })
                } else {
                    this.updateNodes.push({ catId: siblings[i].data.catId, sort: i })
                }

            }
            // 新节点的新层级 请求接口


        },
        // 查找子节点的层级 递归
        updateChildrenNodeLevel(node) {
            if (node.childNodes.length > 0) {
                for (let i = 0; i < node.childNodes.length; i++) {
                    var cNode = node.childNodes[i].data;
                    this.updateNodes.push({ catId: cNode.catId, catLevel: node.childNodes[i].level });
                    this.updateChildrenNodeLevel(node.childNodes[i]);
                }
            }

        },

        // 批量保存提交
        batchSave() {
            this.$http({
                url: this.$http.adornUrl('/product/category/update/sort'),
                method: 'post',
                data: this.$http.adornData(this.updateNodes, false)
            }).then(({ data }) => {
                //success tips
                this.$message({
                    message: "更新成功",
                    type: "success"
                });
                // 刷新菜单
                this.getMenus();
                //删除完维持展开
                this.expandedKey = this.pCid;
                // 避免数据积压
                this.updateNodes = [];
                this.maxLevel = 0;
                // this.pCid = 0;
            })
        },

        // 批量删除
        batchDelete() {
            let catIds = [];
            let checkedNodes = this.$refs.menuTree.getCheckedNodes();
            for (let i = 0; i < checkedNodes.length; i++) {
                catIds.push(checkedNodes[i].catId);
            }
            this.$confirm("是否删除选择内容", '提示', {
                confirmButtonText: '确定',
                cancelButtonText: '取消',
                type: 'warning'
            }).then(() => {
                this.$http({
                    url: this.$http.adornUrl('/product/category/delete'),
                    method: 'post',
                    data: this.$http.adornData(catIds, false)
                }).then(({ data }) => {
                    //success tips
                    this.$message({
                        message: "删除成功",
                        type: "success"
                    });
                    // 刷新菜单
                    this.getMenus();
                    //删除完维持展开

                })
            }).catch(() => {
                this.$message({
                    type: 'info',
                    message: '已取消删除'
                });
            });
        }

    },

    created() {
        this.getMenus();
    }
}
</script>