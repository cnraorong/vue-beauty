<template>
  <ul :class="treeCls">
    <li v-for="(item, index) in data" :key="item.clue" :class="{[`${prefixCls}-treenode-disabled`]: item.disabled,[dropOverCls]: dragOverIndex === index,'filter-node': item.filter, [item.class]: true}" @dragover="dragover" @drop="drop(index,$event)" @dragenter="dragenter(index,$event)" @dragleave="dragleave(index,$event)" ref="node">
      <span :class="[`${prefixCls}-switcher`,{[`${prefixCls}-switcher-disabled`]: item.disabled,[`${prefixCls}-switcher-noop`]: item.isLeaf,[`${prefixCls}-switcher_${item.expanded ? 'open' : 'close'}`]: !item.isLeaf}]" @click="setExpand(item.disabled, index)"></span>
      <span v-if="checkable" :class="checkboxCls(item)" @click.prevent="setCheck(item.disabled || item.disableCheckbox, index)">
        <span :class="prefixCls + '-checkbox-inner'"></span>
      </span>
      <span :title="item.title" :class="selectHandleCls(item)" @click.prevent="setSelect(item.disabled, index)" :draggable="draggable" @dragstart="dragstart(index,$event)" @dragend="dragend">
        <span :class="`${prefixCls}-iconEle ${prefixCls}-icon_loading ${prefixCls}-icon__open`" v-if="item.loading"></span>
        <span :class="prefixCls + '-title'" v-html="item.title"></span>
      </span>
      <transition name="slide-up">
        <tree v-if="!item.isLeaf" :prefix-cls="prefixCls" :data="item.children" :parentNode="item" :pathPrefix="item.fullPath" :clue="`${clue}-${index}`" :multiple="multiple" :checkable="checkable" :class="`${prefixCls}-child-tree-open`" v-show="item.expanded" :draggable="draggable" :async="async"></tree>
      </transition>
    </li>
  </ul>
</template>
<script>
import emitter from '../../mixins/emitter';
import { getOffset } from '../../utils/fn';

export default {
    name: 'Tree',
    mixins: [emitter],
    props: {
        prefixCls: {
            type: String,
            default: 'ant-tree',
        },
        clue: {
            type: String,
            default: '0',
        },
        parentNode: {
            type: Object,
            default: undefined,
        },
        pathPrefix: {
            type: String,
            default: "",
        },
        data: {
            type: Array,
            default: () => [],
        },
        multiple: {
            type: Boolean,
            default: false,
        },
        checkable: {
            type: Boolean,
            default: false,
        },
        draggable: {
            type: Boolean,
            default: false,
        },
        autoLeaf: {
            type: Boolean,
            default: true,
        },
        dragModes: {
            type: Array,
            default: () => [-1, 0, 1],
        },
        canDrop: {
            type: Function,
            default: () => true,
        },
        showLine: {
            type: Boolean,
            default: false,
        },
        async: Function,
    },
    data: () => ({
        dragIndex: -1,
        dragOverIndex: -1,
        dropPosition: 0,
        dragCrossSameTree: false,
    }),
    computed: {
        treeCls() {
            if (this.clue === '0') {
                return [
                    this.prefixCls,
                    { [`${this.prefixCls}-show-line`]: this.showLine },
                ];
            }
            return [
                `${this.prefixCls}-child-tree`,
                { [`${this.prefixCls}-line`]: this.showLine },
            ];
        },
        dropOverCls() {
            let res;
            if(this.dragModes.indexOf(this.dropPosition) == -1) {return res;}
            switch (this.dropPosition) {
                case 0:
                    res = 'drag-over';
                    break;
                case 1:
                    res = 'drag-over-gap-bottom';
                    break;
                case -1:
                    res = 'drag-over-gap-top';
                    break;
                default:
            }
            return res;
        },
    },
    watch: {
        data() {
            this.setKey();
            this.preHandle();
        },
    },
    mounted() {
        this.setKey();
        this.preHandle();

        this.$on('nodeSelected', (params) => {
            if (this.clue !== '0') return this.dispatch('Tree', 'nodeSelected', params);
            if (!this.multiple && params.status) {
                if (this !== params.origin) {
                    for (let i = 0; i < this.data.length; i++) {
                        this.$set(this.data[i], 'selected', false);
                    }
                }
                this.broadcast('Tree', 'cancelSelected', params.origin);
            }
            this.$emit('select', this.getSelectedNodes());
        });

        this.$on('cancelSelected', (ori) => {
            this.broadcast('Tree', 'cancelSelected', ori);

            if (this !== ori) {
                for (let i = 0; i < this.data.length; i++) {
                    this.$set(this.data[i], 'selected', false);
                }
            }
        });

        this.$on('parentChecked', (params) => {
            if (this.clue === params.clue || this.clue.startsWith(`${params.clue}-`)) {
                for (let i = 0; i < this.data.length; i++) {
                    this.$set(this.data[i], 'checked', params.status);
                    this.$set(this.data[i], 'childrenCheckedStatus', params.status ? 2 : 0);
                }
                this.broadcast('Tree', 'parentChecked', params);
            }
        });

        this.$on('childChecked', (params) => {
            if (this.clue === '0') {
                this.$nextTick(() => {
                    this.$emit('check', this.getCheckedNodes());
                });
            }
            if (this === params.origin) return;

            for (const [i, item] of this.data.entries()) {
                if (`${this.clue}-${i}` === params.clue) {
                    const temp = this.getChildrenCheckedStatus(item.children);

                    if (temp !== item.childrenCheckedStatus) {
                        this.$set(this.data[i], 'checked', !!temp);
                        this.$set(this.data[i], 'childrenCheckedStatus', temp);
                    }

                    if (this.clue !== '0') {
                        this.dispatch('Tree', 'childChecked', { origin: this, clue: this.clue });
                    }
                }
            }
        });
        this.$on('dragdrop', (sourceClue, targetClue, dropPosition) => {
            if(this.dragModes.indexOf(this.dropPosition) == -1) {return;}
            var args = Object.assign({}, { sourceClue, targetClue, dropPosition });
            if (this.clue !== '0') return this.dispatch('Tree', 'dragdrop', [sourceClue, targetClue, dropPosition]);
            // 直接父级是否是同一个
            let sourceParent = "0";
            if(sourceClue.indexOf("-")>0) {
                sourceParent = sourceClue.substr(0, sourceClue.lastIndexOf("-"));
            }
            let targetParent = "0";
            if(targetClue.indexOf("-")>0) {
                targetParent = targetClue.substr(0, targetClue.lastIndexOf("-"));
            }
            const sameTree = sourceParent === targetParent;
            sourceClue = sourceClue.split('-');
            let sourceData = this.data;//顶级
            let _sourceData; //拖动的节点Data
            let lastSourceIndex = sourceClue[sourceClue.length - 1] * 1;
            for (let i = 1; i < sourceClue.length - 1; i++) {
                const index = sourceClue[i];
                if (i === 1) {
                    sourceData = sourceData[index];//第一级Item
                } else {
                    sourceData = sourceData.children[index];//第N-1级Item， soure节点的父节点Data
                }
            }
            
            //取得拖动的节点Data
            if (sourceClue.length > 2) {//不是拖动的顶级，这时sourceData是soure节点的父节点Data
                _sourceData = Object.assign({},sourceData.children[lastSourceIndex]);
            } else {//是拖动的顶级,这时sourceData是顶级数组
                _sourceData = Object.assign({},sourceData[lastSourceIndex]);
            }


            targetClue = targetClue.split('-');
            let targetData = this.data;
            const targetIndex = targetClue[targetClue.length - 1] * 1;

            for (let i = 1; i < targetClue.length - 1; i++) {
                const index = targetClue[i];
                if (i === 1) {
                    targetData = targetData[index];
                } else {
                    targetData = targetData.children[index];
                }
            }
            let _targetData; //拖动的节点Data
            if (targetClue.length > 2) {
                _targetData = targetData.children[targetIndex];
            } else {
                _targetData = targetData[targetIndex];
            }

            args = Object.assign(args, { sourceNode: _sourceData, targetNode: _targetData });
            this.$emit('dragdroping', args, () => {
                if (!this.canDrop(_sourceData, _targetData, dropPosition)) return;

                let sourcePositionChange = false;
                switch (dropPosition) {
                    case 0://拖到目标节点里面
                        if (targetClue.length > 2) {
                            targetData = targetData.children[targetIndex];
                        } else {
                            targetData = targetData[targetIndex];
                        }
                        if (targetData.children) {
                            targetData.children.push(_sourceData);
                        } else {
                            this.$set(targetData, 'children', [_sourceData]);
                        }
                        break;
                    case 1://拖到目标节点下方
                    case -1://拖到目标节点上方
                        const p = targetIndex + (dropPosition === -1 ? 0 : dropPosition);
                        if (targetClue.length > 2) {
                            targetData.children.splice(p, 0, _sourceData);
                        } else {
                            targetData.splice(p, 0, _sourceData);
                        }
                        sourcePositionChange = sameTree && p <= lastSourceIndex;
                        break;
                    default:
                }

                if (sourcePositionChange) lastSourceIndex++;
                if (sourceClue.length > 2) {
                    if(this.autoLeaf) {
                        if (sourceData.children.length === 1) {
                            this.$delete(sourceData, 'children');
                        } else {
                            sourceData.children.splice(lastSourceIndex, 1);
                        }
                    }
                    else {
                        sourceData.children.splice(lastSourceIndex, 1);
                    }
                } else {
                    sourceData.splice(lastSourceIndex, 1);
                }
                this.$emit('dragdroped', args);
            });
        });
    },
    methods: {
        dragstart(index, ev) {
            ev.stopPropagation();
            this.dragIndex = index;
            ev.dataTransfer.setData('dragClue', `${this.clue}-${index}`);
        },
        dragover(ev) {
            ev.preventDefault();
            ev.stopPropagation();
        },
        drop(index, ev) {
            ev.stopPropagation();
            this.dragOverIndex = -1;
            const dragClue = ev.dataTransfer.getData('dragClue');
            const selfClue = `${this.clue}-${index}`;

            // 如果拖拽的对象不是自己的父辈级
            if (!selfClue.startsWith(dragClue)) {
                if (this.clue === '0') {
                    this.$emit('dragdrop', dragClue, selfClue, this.dropPosition);
                } else {
                    this.dispatch('Tree', 'dragdrop', [dragClue, selfClue, this.dropPosition]);
                }
            }
        },
        dragenter(index, ev) {
            ev.preventDefault();
            ev.stopPropagation();
            if (this.dragIndex === index) return;
            if (this.dragOverIndex > -1) this.dragCrossSameTree = true;
            this.dragOverIndex = index;
            const offset = getOffset(this.$refs.node[index]);
            const offsetTop = offset.top;
            const offsetHeight = offset.bottom - offset.top;
            const pageY = ev.pageY;
            const gapHeight = 2;

            if (pageY > offsetTop + offsetHeight - gapHeight) {
                this.dropPosition = 1;
            } else if (pageY < offsetTop + gapHeight) {
                this.dropPosition = -1;
            } else {
                this.dropPosition = 0;
            }
            this.$set(this.data[index], 'expanded', true);
        },
        dragleave(index, ev) {
            ev.stopPropagation();
            if (this.dragIndex === index) return;
            if (this.dragCrossSameTree) {
                this.dragCrossSameTree = false;
            } else {
                this.dragOverIndex = -1;
            }
        },
        dragend(ev) {
            ev.stopPropagation();
            this.dragIndex = -1;
        },
        treeNodeCls(item) {
            return {
                [`${this.prefixCls}-treenode-disabled`]: item.disabled,
            };
        },
        checkboxCls(item) {
            return [
                `${this.prefixCls}-checkbox`,
                {
                    [`${this.prefixCls}-checkbox-disabled`]: item.disabled || item.disableCheckbox,
                    [`${this.prefixCls}-checkbox-checked`]: item.checked && item.childrenCheckedStatus === 2,
                    [`${this.prefixCls}-checkbox-indeterminate`]: item.checked && item.childrenCheckedStatus === 1,
                },
            ];
        },
        selectHandleCls(item) {
            const wrap = `${this.prefixCls}-node-content-wrapper`;

            return [
                wrap,
                `${wrap}-normal`,
                {
                    [`${this.prefixCls}-node-selected`]: !item.disable && item.selected,
                    draggable: this.draggable,
                },
            ];
        },
        setKey() {
            for (let i = 0; i < this.data.length; i++) {
                this.data[i].fullPath = `${this.pathPrefix}/${this.data[i].title}`;
                this.data[i].parentNode = this.parentNode;
                this.data[i].clue = `${this.clue}-${i}`;
            }
        },
        preHandle() {
            for (const [i, item] of this.data.entries()) {
                if (!item.children) {
                    this.$set(item, 'isLeaf', true);
                    this.$set(item, 'childrenCheckedStatus', 2);
                    continue;
                }

                this.$set(item, 'isLeaf', false);
                if (item.checked && !item.childrenCheckedStatus) {
                    this.$set(item, 'childrenCheckedStatus', 2);
                    this.broadcast('Tree', 'parentChecked', { status: true, clue: `${this.clue}-${i}` });
                } else {
                    const status = this.getChildrenCheckedStatus(item.children);
                    this.$set(item, 'childrenCheckedStatus', status);

                    if (status !== 0) {
                        this.$set(item, 'checked', true);
                    }
                }
            }
        },
        async setExpand(disabled, index) {
            if (!disabled) {
                const expanded = !this.data[index].expanded;
                this.$set(this.data[index], 'expanded', expanded);
                if(!this.data[index].children) {return;}
                if (expanded && !this.data[index].children.length && this.async) {
                    this.$set(this.data[index], 'loading', true);
                    const data = await this.async(this.data[index]);
                    this.data[index].children = data;
                    this.$set(this.data[index], 'loading', false);
                }
            }
        },
        setSelect(disabled, index) {
            if (!disabled) {
                const selected = !this.data[index].selected;

                if (this.multiple || !selected) {
                    this.$set(this.data[index], 'selected', selected);
                } else {
                    for (let i = 0; i < this.data.length; i++) {
                        if (i === index) {
                            this.$set(this.data[i], 'selected', true);
                        } else {
                            this.$set(this.data[i], 'selected', false);
                        }
                    }
                }

                if (this.clue === '0') {
                    this.$emit('nodeSelected', { origin: this, status: selected });
                } else {
                    this.dispatch('Tree', 'nodeSelected', { origin: this, status: selected });
                }
            }
        },
        setCheck(disabled, index) {
            if (disabled) return;

            const checked = !this.data[index].checked;
            this.$set(this.data[index], 'checked', checked);
            this.$set(this.data[index], 'childrenCheckedStatus', checked ? 2 : 0);
            if (this.clue === '0') {
                this.$emit('childChecked', { origin: this, clue: this.clue });
            } else {
                this.dispatch('Tree', 'childChecked', { origin: this, clue: this.clue });
            }
            this.broadcast('Tree', 'parentChecked', { status: checked, clue: `${this.clue}-${index}` });
        },
        getNodes(data, opt) {
            data = data || this.data;
            let res = [];

            for (const node of data) {
                let tmp = true;
                for (const [key, value] of Object.entries(opt)) {
                    if (node[key] !== value) {
                        tmp = false;
                        break;
                    }
                }
                if (tmp) {
                    res.push(node);
                }
                if (node.children && node.children.length) {
                    res = res.concat(this.getNodes(node.children, opt));
                }
            }
            return res;
        },
        getSelectedNodes() {
            return this.getNodes(this.data, { selected: true });
        },
        getCheckedNodes() {
            return this.getNodes(this.data, { checked: true, childrenCheckedStatus: 2 });
        },
        getHalfCheckedNodes() {
            return this.getNodes(this.data, { checked: true, childrenCheckedStatus: 1 });
        },
        getChildrenCheckedStatus(children) {
            let checkNum = 0;
            let childChildrenAllChecked = true;

            for (const child of children) {
                if (child.checked) {
                    checkNum++;
                }
                if (child.childrenCheckedStatus !== 2) {
                    childChildrenAllChecked = false;
                }
            }
            // 全选
            if (checkNum === children.length) {
                return childChildrenAllChecked ? 2 : 1;
                // 部分选择
            } else if (checkNum > 0) {
                return 1;
            }
            return 0;
        },
        edit(path, action, data) {
            path = path.split('-');
            const isTopNode = path.length === 2;

            let node = this.data;
            const lastIndex = path.pop();

            if (!isTopNode) node = node[path[1]];
            path.splice(0, 2);

            for (const i of path) {
                node = node.children[i];
            }
            switch (action) {
                case 'delete':
                    if (isTopNode) {
                        node.splice(lastIndex, 1);
                    } else {
                        node.children.splice(lastIndex, 1);
                    }
                    break;
                case 'add':
                    let child;
                    if (isTopNode) {
                        child = node[lastIndex];
                    } else {
                        child = node.children[lastIndex];
                    }
                    if (child.children) {
                        child.children.push(data);
                    } else {
                        this.$set(child, 'children', [data]);
                    }
                    break;
                case 'edit':
                    node = isTopNode ? node[lastIndex] : node.children[lastIndex];

                    for (const [key, val] of Object.entries(data)) {
                        node[key] = val;
                    }
                    break;
                default: break;
            }
        },
        editNode(path, data) {
            this.edit(path, 'edit', data);
        },
        addNode(path, data) {
            if (path === '0') return this.data.push(data);
            this.edit(path, 'add', data);
        },
        delNode(path) {
            this.edit(path, 'delete');
        },
    },
};
</script>
