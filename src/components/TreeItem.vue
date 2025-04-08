<template>
  <div class="tree-item" :class="{ 'child-item': path.length > 0 }">
    <div 
      class="tree-item-header"
      :class="{ 
        selected: JSON.stringify(selectedPath) === JSON.stringify([...path, index]),
        'has-children': item.children && item.children.length
      }"
      @click="$emit('select', index, path)"
    >
      <span class="item-name">{{ item.name }}</span>
      <span class="item-type">{{ item.type }}</span>
      <span v-if="item.required" class="item-required">必填</span>
      <span v-if="item.description" class="item-description" :title="item.description">
        <i class="description-icon">i</i>
      </span>
      <button class="remove-btn" @click.stop="$emit('remove', index, path)">删除</button>
    </div>
    
    <!-- 递归渲染子项 -->
    <div v-if="item.children && item.children.length > 0" class="children-container">
      <TreeItem 
        v-for="(child, childIndex) in item.children" 
        :key="`child-${index}-${childIndex}`"
        :item="child" 
        :index="childIndex" 
        :path="[...path, index]" 
        :selectedPath="selectedPath"
        class="nested-item"
        @select="$emit('select', $event, [...path, index])"
        @remove="$emit('remove', $event, [...path, index])"
      />
    </div>
  </div>
</template>

<script>
export default {
  name: 'TreeItem',
  props: {
    item: {
      type: Object,
      required: true
    },
    index: {
      type: Number,
      required: true
    },
    path: {
      type: Array,
      default: () => []
    },
    selectedPath: {
      type: Array,
      default: () => []
    }
  }
}
</script>

<style scoped>
.tree-item {
  margin-bottom: 5px;
}

.tree-item-header {
  display: flex;
  align-items: center;
  padding: 8px;
  border: 1px solid #ddd;
  border-radius: 4px;
  background: #f5f7fa;
  cursor: pointer;
}

.tree-item-header:hover {
  background: #e9ecef;
}

.tree-item-header.selected {
  background: #e1f5fe;
  border-color: #4fc3f7;
}

.tree-item-header.has-children {
  background: #f0f4ff;
  border-color: #b3c6ff;
}

.tree-item-header.has-children.selected {
  background: #daeffe;
  border-color: #4fc3f7;
}

.item-name {
  font-weight: bold;
  margin-right: 10px;
}

.item-type {
  color: #666;
  background: #eee;
  padding: 2px 6px;
  border-radius: 3px;
  font-size: 0.8em;
}

.item-required {
  color: #d32f2f;
  margin-left: 10px;
  font-size: 0.8em;
}

.description-icon {
  display: inline-block;
  width: 16px;
  height: 16px;
  border-radius: 50%;
  background: #64b5f6;
  color: white;
  text-align: center;
  font-size: 0.8em;
  font-style: normal;
  margin-left: 5px;
  line-height: 16px;
}

.item-description {
  cursor: help;
}

.remove-btn {
  margin-left: auto;
  background: #f44336;
  color: white;
  border: none;
  padding: 2px 8px;
  border-radius: 3px;
  cursor: pointer;
  font-size: 0.8em;
}

.remove-btn:hover {
  background: #d32f2f;
}

.children-container {
  margin-left: 20px;
  padding-left: 10px;
  border-left: 2px solid #ddd;
  margin-top: 5px;
}

.nested-item {
  margin-top: 5px;
}
</style>