<template>
  <div>
    <div class="divider"></div>
    <div class="header">
      <el-breadcrumb separator=">">
        <el-breadcrumb-item
          ><span v-show="header.title" class="item" @click="goBack">{{
            header.title
          }}</span></el-breadcrumb-item
        >
        <el-breadcrumb-item v-for="(item, index) in HierarchyList" :key="index"
          ><span class="item" @click="goBack(item)">{{
            item.name
          }}</span></el-breadcrumb-item
        >
      </el-breadcrumb>
    </div>
    <!-- <div class="divider"></div> -->
    <div class="ly-tree-container">
      <tree
        node-key="id"
        :draggable="true"
        :treeData="treeData"
        :props="defaultProps"
        :default-expanded-keys="expandedList"
        :expand-on-click-node="false"
        :render-content="renderContent"
        @more="more"
        @node-drag-start="handleDragStart"
        @node-collapse="collapse"
        @node-expand="expand"
        @dragStart="dragStart"
        @dragOver="dragOver"
        @dragLeave="dragLeave"
        @dragEnd="dragEnd"
      >
      </tree>
    </div>
  </div>

  <contextMenu
    :X="X"
    :Y="Y"
    :menuShow="showMenu"
    @change="changeMenu"
    :menuList="menuList"
  ></contextMenu>
</template>

<script>
import Tree from "@/components/newTree/tree.vue";
import contextMenu from "@/components/context_menu.vue";
import { store } from "@/store";
import { mapState } from "vuex";
const { ipcRenderer } = require("electron");
const dayjs = require("dayjs");
import { getQueryByName } from "@/utils";

// import mitt from "mitt";

// const mittExample = mitt();
let timeout;
export default {
  components: {
    Tree,
    contextMenu,
  },
  computed: {
    ...mapState("header", {
      header: (state) => state.note,
      headerClose: (state) => state.close,
    }),
  },
  data() {
    return {
      X: 0,
      Y: 0,
      showMenu: false,
      menuList: [
        {
          key: 0,
          name: "完成",
        },
        // {
        //   key: 1,
        //   name: "编辑描述",
        // },
        {
          key: 2,
          name: "删除",
        },
      ],
      tree: [],
      expandedList: [],
      treeData: [],
      defaultProps: {
        children: "child",
        label: "name",
      },
      HierarchyList: [],
      cursorPosition: -1,
      currentNode: {},
      dragSortObj: {
        dragElement: {},
        dragNode: {},
        targetNode: {},
        targetElement: {},
        insertionType: 0, //象目标节点插入方向：0.向后插入，1.插为子节点,2.向前插入
      },
      isCNInput: false,
    };
  },
  props: {
    currentItem: {
      type: Object,
      default() {
        return {};
      },
    },
    setPageTypeText: {
      type: String,
      default: "outline",
    },
  },
  watch: {
    setPageTypeText: function () {
      store.dispatch("header/setPageTypeText", this.setPageTypeText);
    },
    currentItem: {
      deep: true,
      handler() {
        this.init();
      },
    },
    headerClose: {
      deep: true,
      handler(val) {
        if (val) return;
        this.saveTree();
      },
    },
  },
  async created() {
    store.dispatch("header/setPageTypeText", this.setPageTypeText);
    await this.init();

    window.addEventListener("click", () => {
      this.showMenu = false;
    });
    window.addEventListener("keydown", (e) => {
      let keyCode = e.keyCode;
      const that = this;
      if (e.key === "s" && keyCode === 83) {
        const fun = function () {
          that.saveTree("save");
          that.$message({
            message: "保存成功",
            type: "success",
            duration: 1000,
          });
        };

        that.debounce(fun, 500, true)();
      }
    });
  },

  methods: {
    debounce(func, wait, immediate) {
      return () => {
        let context = this;
        let args = arguments;

        if (timeout) clearTimeout(timeout);
        if (immediate) {
          let callNow = !timeout;
          timeout = setTimeout(function () {
            timeout = null;
          }, wait);
          if (callNow) func.apply(context, args);
        } else {
          timeout = setTimeout(function () {
            func.apply(context, args);
          }, wait);
        }
      };
    },
    dragEnd() {
      // console.log("结束", event, that.node);
      let { targetNode, dragNode, targetElement, insertionType } =
        this.dragSortObj;
      // console.log("拖动对象", dragNode);
      // console.log("目标对象", targetObj);

      let targetObjParentList = targetNode.parent.data;
      let dataParent = dragNode.parent.data;
      if (!Array.isArray(targetObjParentList)) {
        if (targetObjParentList.id === dragNode.data.id) return;
        targetObjParentList = targetObjParentList.child;
      }
      if (!Array.isArray(dataParent)) {
        dataParent = dataParent.child;
      }
      let currentIndex = targetObjParentList.findIndex(
        (item) => item.id === targetNode.data.id
      );
      let dataIndex = dataParent.findIndex(
        (item) => item.id === dragNode.data.id
      );
      // debugger;
      let targetLevel = targetNode.data.level;
      if (dragNode.data.level === targetNode.data.level) {
        if (insertionType === 1) {
          dragNode.data.level = targetLevel + 1;
          targetNode.data.child.push(dragNode.data);
          dataParent.splice(dataIndex, 1);
        } else {
          dragNode.data.level = targetLevel;
          // if (currentIndex > 0 && insertionType === 2) {
          //   currentIndex--;
          // }
          let removeIndex = targetObjParentList.findIndex(
            (item) => item.id === dragNode.data.id
          );
          targetObjParentList.splice(removeIndex, 1);
          targetObjParentList.splice(currentIndex, 0, dragNode.data);

          // dataParent.splice(dataIndex, 1, targetNode.data);
        }
      } else {
        dragNode.data.level = targetLevel;
        // if (insertionType === 2) {
        //   // currentIndex += 1;
        // }
        targetObjParentList.splice(currentIndex, 0, dragNode.data);
        dataParent.splice(dataIndex, 1);
      }
      this.treeData = JSON.parse(JSON.stringify(this.treeData));
      this.updateTree();
      for (let item of targetElement.children) {
        if (item.getAttribute("class") === "line") {
          item.style.display = "none";
          item.style.marginLeft = "0px";
        }
      }
    },
    dragLeave(event) {
      for (let item of event.currentTarget.children) {
        if (item.getAttribute("class") === "line") {
          item.style.display = "none";
          item.style.marginLeft = "0px";
        }
      }
    },
    dragOver(event, that) {
      const node = that.node;
      this.dragSortObj.targetNode = node;
      this.dragSortObj.targetElement = event.currentTarget;

      const { dragElement, dragNode } = this.dragSortObj;
      if (!Array.isArray(node.parent.data) && dragNode) {
        console.log(node.parent.data.id === dragNode.data.id);
        if (node.parent.data.id === dragNode.data.id) {
          event.dataTransfer.effectAllowed = "none";
          return;
        }
      }
      // console.log("目标", event.layerY, "开始拖拽", dragElement.layerY);
      for (let item of event.currentTarget.children) {
        if (item.getAttribute("class") === "line") {
          item.style.display = "block";
          if (dragElement.layerY > event.layerY) {
            //在前面插入
            item.style.top = "-2px";
            this.dragSortObj.insertionType = 2;
          } else {
            item.style.top = "21px";
          }
          if (event.clientX - dragElement.clientX > 30) {
            item.style.marginLeft = "35px";
            this.dragSortObj.insertionType = 1;
          } else {
            item.style.marginLeft = "0px";
          }
        }
      }
    },
    dragStart(event, that) {
      this.dragSortObj.dragElement = event;
      this.dragSortObj.dragNode = that.node;
    },
    async init() {
      let winId = getQueryByName("winId");
      let note = {};
      if (winId === "undefined") {
        note._id = this.$createdId();
      } else {
        note = JSON.parse(JSON.stringify(this.currentItem));
        if (JSON.stringify(note) === "{}") {
          note = await ipcRenderer.invoke("getNote", winId);
        }
        this.expandedList = JSON.parse(JSON.stringify(note.expandedList || []));
        this.tree = JSON.parse(JSON.stringify(note.tree || []));
        if (note.title) {
          store.dispatch("header/setIsEditedTitle", false);
        }
      }
      if (!note.tree) {
        let id = this.$createdId();
        this.tree.push({
          id,
          name: "",
          level: 1,
          child: [],
        });
        this.refresh();
        this.$nextTick(() => {
          let div = document.getElementById(`${id}`);
          // console.log("div", div);
          div.focus();
        });
      } else {
        this.refresh();
      }
      delete note.tree;
      delete note.expandedList;
      store.dispatch("header/setNote", note || {});
    },
    saveTree(typeText) {
      let { timing } = this.header;
      const note = this.header;
      const tree = this.tree;
      const expandedList = this.expandedList;
      // // const text = this.editor.getText();
      let tempOjb = { tree, expandedList, ...note };
      tempOjb.title = tempOjb.title || "无标题";
      tempOjb.modeType = 1;
      if (timing) {
        tempOjb.timing = timing;
        tempOjb.timinGtimeStamp = dayjs(timing).valueOf();
        tempOjb.timingStatus = 0;
      }

      ipcRenderer.send(
        "closeEdited",
        note._id,
        JSON.parse(JSON.stringify(tempOjb)),
        typeText
      );
      if (typeText != "save") {
        store.dispatch("header/setHeaderClose", false);
      }
    },
    changeMenu(key) {
      let currentNode = this.currentNode;
      switch (key) {
        case 0:
          {
            currentNode.data.isComplete = currentNode.data.isComplete
              ? false
              : true;
          }

          break;
        case 2:
          {
            let parentChild = currentNode.parent.data.child;
            const index = parentChild.findIndex(
              (item) => item.id === currentNode.data.id
            );
            parentChild.splice(index, 1);
          }

          break;
        default:
          break;
      }
      this.updateTree();
      this.showMenu = false;
    },
    more(node) {
      // console.log(node);
      this.X = event.clientX;
      this.Y = event.clientY + 20;
      this.currentNode = node;
      const data = node.data;
      if (data.isComplete) {
        this.menuList[0].name = "取消完成";
      } else {
        this.menuList[0].name = "完成";
      }
      this.showMenu = !this.showMenu;
    },
    collapse(data, node) {
      this.findNodeId(data, false, node);
    },
    expand(data) {
      this.findNodeId(data, true);
    },
    findNodeId(data, bool) {
      // console.log("idList", idList);
      // if (!idList.length) return;
      // console.log("node", node);
      if (bool) {
        this.expandedList.push(data.id);
      } else {
        let idList = [];
        const fn = function (array) {
          if (array.length) {
            for (let item of array) {
              idList.push(item.id);
              if (item.child.length) {
                fn(item.child);
              }
            }
          }
          return;
        };
        fn(data.child);
        idList.forEach((id) => {
          let i = this.expandedList.findIndex((item) => item === id);
          this.expandedList.splice(i, 1);
        });
      }
      this.updateTree();
    },
    handleDragStart(node) {
      console.log("drag start", node);
    },
    refresh() {
      this.treeData = JSON.parse(JSON.stringify(this.tree));
    },
    customFocus(node, len) {
      console.log("node", node, len);
      console.log("光标", this.cursorPosition);
      if (typeof node === "undefined") return;
      let range = document.createRange();
      let sel = window.getSelection();
      if (len > 0) {
        if (len > this.cursorPosition) {
          len = this.cursorPosition + 1;
        }
        if (len < this.cursorPosition) {
          len = this.cursorPosition - 1;
        }
      }
      console.log("光标位置", len);
      range.setStart(node, len);
      range.collapse(true);
      sel.removeAllRanges();
      sel.addRange(range);
    },
    getName(data) {
      if (this.isCNInput) return;
      this.inputNodeName(event, data);
    },
    compositionstart() {
      this.isCNInput = true;
    },
    compositionend(data) {
      this.isCNInput = false;
      this.cursorPosition = event.currentTarget.innerText.length;
      this.inputNodeName(event, data);
    },
    inputNodeName(e, data) {
      let value = e.currentTarget.innerText;
      data.name = value;
      e.currentTarget.innerText = value;
      this.updateTree();
      this.$nextTick(() => {
        this.customFocus(e.currentTarget.childNodes[0], value.length);
      });
    },
    goBack(data) {
      if (!data.child) {
        this.refresh();
        this.HierarchyList = [];
        return;
      }
      let index = this.HierarchyList.findIndex((item) => item.id === data.id);
      if (index >= 0) {
        this.HierarchyList = this.HierarchyList.slice(0, index + 1);
      }
      this.treeData = data.child;
    },
    toChild(data) {
      let index = this.HierarchyList.findIndex((item) => item.id === data.id);
      if (index === -1) {
        this.HierarchyList.push(data);
        let child = data.child;

        if (!child.length) {
          child = [
            {
              child: [],
              name: "",
            },
          ];
        }
        this.treeData = child;
      }
    },
    mouseover(node) {
      if (node.expanded || !node.childNodes.length) {
        event.currentTarget.style.background = "#cfcfcf";
      }
    },
    mouseout(node) {
      if (node.expanded || !node.childNodes.length) {
        event.currentTarget.style.background = "#fff";
      }
    },
    updateTree() {
      let tree = JSON.parse(JSON.stringify(this.treeData));
      this.tree = tree;
      this.$emit("onChangeTree", tree, this.expandedList);
    },
    shortcutKey(node, data) {
      const keyText = event.key;

      const cursor = this.getCaretPosition(event.currentTarget);

      if (
        ["Enter", "Tab", "Backspace", "ArrowUp", "ArrowDown"].includes(keyText)
      ) {
        if (["Enter", "Tab", "ArrowUp", "ArrowDown"].includes(keyText)) {
          event.preventDefault();
        }

        if (keyText === "Backspace") {
          if (data.name.length) return;
        }
        const parent = node.parent;
        const expanded = node.expanded;
        // console.log("node", node);

        let parentData = parent.data;
        let newObj = {
          id: this.$createdId(),
          level: parent.level + 1,
          child: [],
          name: "",
        };
        if (data && keyText != "Enter") {
          newObj = JSON.parse(JSON.stringify(data));
        }
        // console.log("parentData ", parentData);
        let parentChild = parentData.child;
        if (!parentChild && Array.isArray(parentData)) {
          parentChild = parentData;
        }
        let index = parentChild.findIndex((item) => item.id === data.id);
        let expandedList = this.expandedList;
        // debugger;
        if (keyText === "Enter") {
          if (data.level === 1) newObj.level = 1;

          if (data.level === 1 && (!cursor || !expanded)) {
            let insertIndex = this.treeData.findIndex(
              (item) => item.id === data.id
            );
            if (insertIndex === -1) insertIndex = 0;
            if (cursor) {
              insertIndex++;
            } else if (insertIndex > 0) {
              // insertIndex--;
            }
            this.treeData.splice(insertIndex, 0, newObj);
          } else {
            if (!node.childNodes.length) {
              if (cursor) {
                index++;
              }
              parentChild.splice(index, 0, newObj);
            } else {
              index = data.child.findIndex((item) => item.id === data.id);
              if (index >= 0) {
                data.child.splice(index, 0, newObj);
              } else {
                data.child.unshift(newObj);
              }
            }
          }
          expandedList.push(newObj.id);
        } else if (keyText === "Tab") {
          const shiftKey = event.shiftKey;
          if (parent.childNodes.length > 1 && !shiftKey) {
            newObj.level++;
            if (index === -1) {
              parentChild = this.treeData;
              index = parentChild.findIndex((item) => item.id === data.id);
            }
            parentChild.splice(index, 1);
            let childObj = {};
            if (index > 0) {
              // console.log("childObj", childObj, index);
              childObj = parentChild[index - 1];
              childObj.child.splice(index, 0, newObj);
            }
          } else if (shiftKey) {
            index = parentChild.findIndex((item) => item.id === data.id);
            parentChild.splice(index, 1);
            newObj.level--;
            if (parent.parent && !Array.isArray(parent.parent.data)) {
              parent.parent.data.child.push(newObj);
            } else if (parent && parent.data) {
              const treeDataIndex = this.treeData.findIndex(
                (item) => item.id === data.id
              );
              if (treeDataIndex === -1) {
                this.treeData.push(newObj);
              }
            }
          }
        } else if (
          keyText === "Backspace" &&
          !data.name.length &&
          !data.child.length
        ) {
          event.preventDefault();
          // debugger;
          if (data.level === 1) {
            const treeDataIndex = this.treeData.findIndex(
              (item) => item.id === data.id
            );
            this.treeData.splice(treeDataIndex, 1);
            if (treeDataIndex > 0) {
              newObj = this.treeData[treeDataIndex - 1];
            }
            if (!this.treeData.length) {
              this.treeData.push(newObj);
            }
          } else {
            parentChild.splice(index, 1);
            let lastNodeObj = parentChild[parentChild.length - 1];
            if (lastNodeObj) {
              newObj = lastNodeObj;
            } else {
              newObj = JSON.parse(JSON.stringify(parentData));
            }
          }
          this.cursorPosition = newObj.name.length;
          // debugger;
          let expandedIndex = expandedList.findIndex(
            (item) => item === data.id
          );
          if (expandedIndex >= 0) {
            expandedList.splice(expandedIndex, 1);
          }
        } else if (["ArrowUp", "ArrowDown"].includes(keyText)) {
          event.stopImmediatePropagation();
          // debugger;
          if (keyText === "ArrowUp") {
            if (index >= 1) {
              newObj = parentChild[index - 1];
            } else {
              newObj = parentData;
            }
            this.cursorPosition = newObj.name.length;
          } else {
            if (index >= 0 && parentChild.length - 1 > index) {
              newObj = parentChild[index + 1];
            } else {
              if (newObj.child && newObj.child.length) {
                newObj = newObj.child[0];
              }
            }
          }
        }
        this.treeData = JSON.parse(JSON.stringify(this.treeData));
        this.updateTree();
        const that = this;
        let sel = window.getSelection();
        sel.removeAllRanges();
        that.$nextTick(() => {
          let id = newObj.id;
          let div = document.getElementById(`${id}`);
          // console.log(div);
          if (div) {
            // div.focus();
            div.addEventListener(
              "focus",
              (e) => {
                // console.log("触发聚焦");
                e.preventDefault();
              },
              {
                capture: false,
                once: true,
              }
            );

            let focus = new Event("focus");
            div.dispatchEvent(focus);

            setTimeout(() => {
              that.customFocus(div.childNodes[0], newObj.name.length);
            }, 200);
          }
        });
      }
    },
    getCaretPosition(editableDiv) {
      let caretPos = 0,
        sel,
        range;
      if (window.getSelection) {
        sel = window.getSelection();
        if (sel.rangeCount) {
          range = sel.getRangeAt(0);
          if (range.commonAncestorContainer.parentNode == editableDiv) {
            caretPos = range.endOffset;
          }
        }
      } else if (document.selection && document.selection.createRange) {
        range = document.selection.createRange();
        if (range.parentElement() == editableDiv) {
          var tempEl = document.createElement("span");
          editableDiv.insertBefore(tempEl, editableDiv.firstChild);
          var tempRange = range.duplicate();
          tempRange.moveToElementText(tempEl);
          tempRange.setEndPoint("EndToEnd", range);
          caretPos = tempRange.text.length;
        }
      }

      this.cursorPosition = caretPos;
      return caretPos;
    },
    nodeStyle(data) {
      let result = {
        width: "47vw",
        outline: "none",
        marginLeft: "10px",
        marginTop: "3px",
      };
      if (data.isComplete) {
        result = Object.assign(result, {
          textDecorationLine: "line-through",
          color: "#b8b8b8",
          textDecorationColor: " #878787",
          textDecorationThickness: "2px",
        });
      }
      return result;
    },

    renderContent(h, { node, data }) {
      // console.log("node", node, "data", data, _self);
      data = data || {};
      const nodeStyle = this.nodeStyle(data);
      return (
        <div onClick={() => (this.showMenu = false)} class="node_main">
          <div class="ly-tree-node">
            <li
              onClick={() => this.toChild(data)}
              onMouseover={() => this.mouseover(node)}
              onMouseout={() => this.mouseout(node)}
              style={{
                background: node.isLeaf || node.expanded ? "#fff" : "#cfcfcf",
              }}
            ></li>
            <div
              id={data.id}
              onCompositionstart={() => this.compositionstart(data)}
              onCompositionend={() => this.compositionend(data)}
              onInput={() => this.getName(data)}
              style={nodeStyle}
              contenteditable="true"
              onKeydown={() => this.shortcutKey(node, data)}
            >
              {data.name}
            </div>
          </div>
        </div>
      );
    },
  },
};
</script>
<style  lang="scss" scoped>
.header {
  margin-left: 20px;
  margin-bottom: 15px;
  margin-top: 20px;
  .item {
    box-shadow: 0 0 4px #cbcbcb;
    padding: 7px;
    border-radius: 5px;
    cursor: pointer;
  }
  span {
    font-size: 13px;
    font-weight: 600;
    margin-bottom: 20px;
  }
}
</style>
