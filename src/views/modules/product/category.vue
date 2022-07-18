<!--  -->
<template>
    <div>
        <el-switch v-model="draggable" active-text="开启拖拽" inactive-text="关闭拖拽"></el-switch>
        <el-button v-if="draggable" @click="batchSave">批量保存</el-button>
        <el-button type="danger" @click="batchDelete">批量删除</el-button>
        <el-tree :data="menus" show-checkbox node-key="catId" :props="defaultProps" @node-click="handleNodeClick"
            :expand-on-click-node="false" :default-expanded-keys="expandedKey" :draggable="draggable"
            :allow-drop="allowDrop" @node-drop="handleDrop" ref="menuTree">
            <span class="custom-tree-node" slot-scope="{ node, data }">
                <span>{{ node.label }}</span>
                <span>
                    <el-button v-if="node.level <= 2" type="text" size="mini" @click="() => append(data)">
                        Append
                    </el-button>
                    <el-button type="text" size="mini" @click="() => edit(data)">
                        Edit
                    </el-button>
                    <el-button v-if="node.childNodes.length == 0" type="text" size="mini"
                        @click="() => remove(node, data)">
                        Delete
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
                    <el-input v-model="category.icon"></el-input>
                </el-form-item>
                <el-form-item label="计量单位">
                    <el-input v-model="category.productUnit"></el-input>
                </el-form-item>
            </el-form>
            <span slot="footer" class="dialog-footer">
                <el-button @click="dialogVisible = false">取 消</el-button>
                <el-button type="primary" @click="submitData">确 定</el-button>
            </span>
        </el-dialog>
    </div>
</template>

<script>
export default {
    //import引入的组件需要注入到对象中才能使用
    components: {},
    data() {
        return {
            updateNodes: [],
            title: "", //弹出框标题
            dialogType: "", //append or edit
            maxLevel: 1,
            draggable: false,// 是否可拖拽
            category: { catId: null, name: "", parentCid: 0, catLevel: 0, showStatus: 1, sort: 0, icon: "", productUnit: "" },
            dialogVisible: false,
            expandedKey: [],
            menus: [],
            defaultProps: {
                children: 'children',
                label: 'name'
            }
        };
    },
    //监听属性 类似于data概念
    computed: {},
    //监控data中的数据变化
    watch: {},
    //方法集合
    methods: {
        //添加菜单项dialog
        append(data) {
            console.log("append", data);
            this.dialogVisible = true;
            this.dialogType = "append";
            this.title = "添加分类";
            this.category.parentCid = data.catId;
            this.category.catLevel = data.catLevel + 1;
        },
        //编辑菜单项
        edit(data) {
            console.log("要修改的菜单：", data);
            this.dialogVisible = true;
            this.dialogType = "edit";
            this.title = "修改分类";
            //获取最新数据
            this.$http({
                url: this.$http.adornUrl(`/product/category/info/${data.catId}`),
                method: 'get'
            }).then(({ data }) => {
                //请求成功
                this.category.name = data.data.name;
                this.category.catId = data.data.catId;
                this.category.icon = data.data.icon;
                this.category.productUnit = data.data.productUnit;
                this.category.parentCid = data.data.parentCid;
            })

        },
        //
        submitData() {
            console.log("三级分类数据：", this.category);
            if (this.dialogType == "append") {
                this.$http({
                    url: this.$http.adornUrl('/product/category/save'),
                    method: 'post',
                    data: this.$http.adornData(this.category, false)
                }).then(({ data }) => {
                    this.$message({
                        type: 'success',
                        message: '菜单保存成功!'
                    });
                    //关闭对话框
                    this.dialogVisible = false;
                    // 刷新菜单
                    this.getMenus();
                    // 展开父菜单
                    this.expandedKey = [this.category.parentCid];
                    this.category.parentCid = 0;
                    this.category.catLevel = 0;
                });
            } else if (this.dialogType == "edit") {
                let { catId, name, icon, productUnit } = this.category;
                let data = { catId, name, icon, productUnit };
                this.$http({
                    url: this.$http.adornUrl('/product/category/update'),
                    method: 'post',
                    data: this.$http.adornData(data, false)
                }).then(({ data }) => {
                    this.$message({
                        type: 'success',
                        message: '菜单修改成功!'
                    });
                    //关闭对话框
                    this.dialogVisible = false;
                    // 刷新菜单
                    this.getMenus();
                    // 展开父菜单
                    this.expandedKey = [this.category.parentCid];
                    this.category.name = "";
                    this.category.catId = null;
                    this.category.icon = "";
                    this.category.productUnit = "";
                    this.category.parentCid = 0;
                });
            }
        },
        //删除菜单项
        remove(node, data) {
            console.log("remove", node, data);
            var ids = [data.catId];
            //消息提示
            this.$confirm(`是否删除【${data.name}】菜单?`, '提示', {
                confirmButtonText: '确定',
                cancelButtonText: '取消',
                type: 'warning'
            }).then(() => {
                this.$http({
                    url: this.$http.adornUrl('/product/category/delete'),
                    method: 'post',
                    data: this.$http.adornData(ids, false)
                }).then(({ data }) => {
                    this.$message({
                        type: 'success',
                        message: '菜单删除成功!'
                    });
                    this.getMenus();
                    this.expandedKey = [node.parent.data.catId];
                });
            }).catch(() => { });
        },
        //
        allowDrop(draggingNode, dropNode, type) {
            //console.log("allowDrop: ", draggingNode, dropNode, type);
            this.countNodeLevel(draggingNode);
            let deep = this.maxLevel - draggingNode.level + 1;
            //console.log("maxLevel:", this.maxLevel);
            //console.log("deep:", deep);
            this.maxLevel = 1;
            if (type == "inner") {
                return (deep + dropNode.level) <= 3;
            } else {
                return (deep + dropNode.parent.level) <= 3;
            }
        },
        // 统计当前节点总层数
        countNodeLevel(node) {
            if (node.childNodes != null && node.childNodes.length > 0) {
                for (let i = 0; i < node.childNodes.length; ++i) {
                    if (node.childNodes[i].level > this.maxLevel) {
                        this.maxLevel = node.childNodes[i].level;
                    }
                    this.countNodeLevel(node.childNodes[i]);
                }
            } else {
                this.maxLevel = node.level > this.maxLevel ? node.level : this.maxLevel;
            }
        },
        //刷新
        getMenus() {
            this.$http({
                url: this.$http.adornUrl('/product/category/list/tree'),
                method: 'get'
            }).then(({ data }) => {
                this.menus = data.data;
            })
        },
        handleDrop(draggingNode, dropNode, dropType, ev) {
            console.log('tree drop: ', draggingNode, dropNode, dropType);
            //当前节点最新父节点id
            let pCid = 0;
            let siblings = null;
            if (dropType == "inner") {
                pCid = dropNode.data.catId;
                siblings = dropNode.childNodes;
            } else {
                pCid = dropNode.data.parentCid;
                siblings = dropNode.parent.childNodes;
            }
            // 当前节点最新顺序
            for (let i = 0; i < siblings.length; ++i) {
                if (siblings[i].data.catId == draggingNode.data.catId) {
                    let catLevel = draggingNode.level;
                    if (siblings[i].level != draggingNode.level) {
                        // 层级发生变化
                        catLevel = siblings[i].level;
                        // 递归修改子节点层级
                        this.updateChildNodeLevel(siblings[i]);
                    }
                    this.updateNodes.push({ catId: siblings[i].data.catId, sort: i, parentCid: pCid, catLevel: catLevel });
                } else {
                    this.updateNodes.push({ catId: siblings[i].data.catId, sort: i });
                }
            }
            console.log("updateNodes:", this.updateNodes);
        },
        // clickEvent 批量保存
        batchSave() {
            // 发送
            this.$http({
                url: this.$http.adornUrl('/product/category/update/sort'),
                method: 'post',
                data: this.$http.adornData(this.updateNodes, false)
            }).then(({ data }) => {
                this.$message({
                    type: 'success',
                    message: '菜单列表更新成功!'
                });
                this.getMenus();
                this.updateNodes = [];
                this.maxLevel = 1;
            });
        },
        //
        updateChildNodeLevel(node) {
            if (node.childNodes.length > 0) {
                for (let i = 0; i < node.childNodes.length; ++i) {
                    var cNode = node.childNodes[i].data;
                    this.updateNodes.push({ catId: cNode.catId, catLevel: node.childNodes[i].level });
                    this.updateChildNodeLevel(node.childNodes[i]);
                }
            }
        },
        //批量删除
        batchDelete() {
            let catIds = [];
            let checkedNodes = this.$refs.menuTree.getCheckedNodes();
            console.log("checkedNodes:", checkedNodes);
            checkedNodes.forEach((data) => {
                catIds.push(data.catId);
            })
            console.log(catIds)
            //消息提示
            this.$confirm(`是否批量删除菜单?`, '提示', {
                confirmButtonText: '确定',
                cancelButtonText: '取消',
                type: 'warning'
            }).then(() => {
                this.$http({
                    url: this.$http.adornUrl('/product/category/delete'),
                    method: 'post',
                    data: this.$http.adornData(catIds, false)
                }).then(({ data }) => {
                    this.$message({
                        type: 'success',
                        message: '菜单批量删除成功!'
                    });
                    this.getMenus();
                });
            }).catch(() => { });
        }
    },
    //生命周期 - 创建完成（可以访问当前this实例）
    created() {
        this.getMenus();
    },
    //生命周期 - 挂载完成（可以访问DOM元素）
    mounted() {

    },
    beforeCreate() { }, //生命周期 - 创建之前
    beforeMount() { }, //生命周期 - 挂载之前
    beforeUpdate() { }, //生命周期 - 更新之前
    updated() { }, //生命周期 - 更新之后
    beforeDestroy() { }, //生命周期 - 销毁之前
    destroyed() { }, //生命周期 - 销毁完成
    activated() { }, //如果页面有keep-alive缓存功能，这个函数会触发
}
</script>
<style >
/*@import url(); 引入公共css类*/
</style>