<script setup>
import { ref, watch, computed } from 'vue'
import Ajv from 'ajv'
import * as z from 'zod'
import { zodResponseFormat } from 'openai/helpers/zod'
import TreeItem from './components/TreeItem.vue'
import JsonViewer from './components/JsonViewer.vue'

// 初始化JSON schema生成器
const ajv = new Ajv({ allErrors: true })

// 定义状态
const jsonStructure = ref([
  { 
    name: 'user',
    type: 'object',
    required: true,
    description: 'user information',
    children: [
      { name: 'name', type: 'string', required: true },
      { name: 'age', type: 'number', required: false }
    ]
  }
])

// 选中的元素索引
const selectedIndex = ref(null)
const selectedPath = ref([])
const currentEditingItem = ref(null) // 当前正在编辑的项目

// Tab 切换控制
const activeTab = ref('zod')

// 表单属性
const newItemName = ref('')
const newItemType = ref('string')
const newItemRequired = ref(true)
const newItemDescription = ref('')
const enumValues = ref('') // 添加枚举值输入

// 计算生成的JSON Schema
const generatedSchema = computed(() => {
  return generateJSONSchema(jsonStructure.value)
})

// 生成用于OpenAI structured output的完整Schema
const openaiSchema = computed(() => {
  const baseSchema = generatedSchema.value
  return {
    type: "object",
    properties: baseSchema.properties,
    required: baseSchema.required || []
  }
})

// 生成Zod Schema对象
const zodSchema = computed(() => {
  try {
    const schema = generateZodSchema(jsonStructure.value)
    return schema ? z.object(schema) : null
  } catch (e) {
    console.error("Error in zodSchema computed property:", e)
    return null
  }
})

// 生成用于OpenAI的zodResponseFormat结果
const zodOpenaiSchema = computed(() => {
  try {
    if (!zodSchema.value) return null
    const result = zodResponseFormat(zodSchema.value, "data")
    return result['json_schema']
  } catch (e) {
    console.error("Error generating zodResponseFormat:", e)
    return {
      error: "生成Zod Schema时出错，请检查JSON结构是否有效",
      details: e.message
    }
  }
})

// 将结构转换为Zod Schema
function generateZodSchema(structure) {
  if (!structure || structure.length === 0) return null
  
  const schemaProps = {}
  
  structure.forEach(item => {
    try {
      if (item.type === 'object') {
        if (item.children && item.children.length > 0) {
          const childSchema = generateZodSchema(item.children)
          if (childSchema) {
            schemaProps[item.name] = item.required 
              ? z.object(childSchema) 
              : z.object(childSchema).optional()
          } else {
            schemaProps[item.name] = item.required 
              ? z.object({}) 
              : z.object({}).optional()
          }
        } else {
          schemaProps[item.name] = item.required 
            ? z.object({}) 
            : z.object({}).optional()
        }
      } else if (item.type.startsWith('array[')) {
        const arrayType = item.type.match(/array\[(.*?)\]/)[1]
        if (arrayType === 'object') {
          if (item.children && item.children.length > 0) {
            const childSchema = generateZodSchema(item.children)
            if (childSchema) {
              const arrayItems = z.object(childSchema)
              schemaProps[item.name] = item.required 
                ? z.array(arrayItems) 
                : z.array(arrayItems).optional()
            } else {
              schemaProps[item.name] = item.required 
                ? z.array(z.object({})) 
                : z.array(z.object({})).optional()
            }
          } else {
            schemaProps[item.name] = item.required 
              ? z.array(z.object({})) 
              : z.array(z.object({})).optional()
          }
        } else {
          var arrayZodType = getZodTypeForPrimitive(arrayType)
          if (arrayType === 'enum') {
            const values = item.enumValues.split(',').map(v => v.trim()).filter(v => v)
            if (values.length > 0) {
                arrayZodType = item.required 
              ? z.enum(values) 
              : z.enum(values).optional()
            } else {
              // 如果没有有效的枚举值，则使用字符串类型代替
              arrayZodType = item.required 
                ? z.string() 
                : z.string().optional()
            }
          }
          schemaProps[item.name] = item.required 
            ? z.array(arrayZodType) 
            : z.array(arrayZodType).optional()
        }
      } else if (item.type === 'enum' && item.enumValues) {
        const values = item.enumValues.split(',').map(v => v.trim()).filter(v => v)
        if (values.length > 0) {
          schemaProps[item.name] = item.required 
            ? z.enum(values) 
            : z.enum(values).optional()
        } else {
          // 如果没有有效的枚举值，则使用字符串类型代替
          schemaProps[item.name] = item.required 
            ? z.string() 
            : z.string().optional()
        }
      } else {
        const zodType = getZodTypeForPrimitive(item.type)
        let schema = zodType
        if (item.description) {
          schema = schema.describe(item.description)
        }
        schemaProps[item.name] = item.required ? schema : schema.optional()
      }
    } catch (e) {
      console.error(`Error processing item ${item.name}:`, e)
      // 使用简单字符串作为后备
      schemaProps[item.name] = item.required 
        ? z.string().describe(`Error: ${e.message}`) 
        : z.string().optional().describe(`Error: ${e.message}`)
    }
  })
  
  return schemaProps
}

// 获取基本类型对应的Zod类型
function getZodTypeForPrimitive(type) {
  switch(type) {
    case 'string': return z.string()
    case 'number': return z.number()
    case 'integer': return z.number().int()
    case 'boolean': return z.boolean()
    case 'enum': 
      const values = enumValues.value.split(',').map(v => v.trim()).filter(v => v)
      return values.length > 0 ? z.enum(values) : z.string()
    default: return z.string()
  }
}

// 将结构转换为JSON Schema
function generateJSONSchema(structure, parentPath = '') {
  const schema = {
    type: "object",
    properties: {},
    required: []
  }
  
  structure.forEach(item => {
    if (item.type === 'object') {
      const childSchema = generateJSONSchema(item.children || [], `${parentPath}${item.name}.`)
      schema.properties[item.name] = {
        type: "object",
        properties: childSchema.properties
      }
      
      if (childSchema.required && childSchema.required.length) {
        schema.properties[item.name].required = childSchema.required
      }
      
      if (item.description) {
        schema.properties[item.name].description = item.description
      }
      
      if (item.required) {
        schema.required.push(item.name)
      }
    } else if (item.type.startsWith('array[')) {
      const arrayType = item.type.match(/array\[(.*?)\]/)[1]
      
      if (arrayType === 'object') {
        if (item.children && item.children.length) {
          // 使用子元素定义对象数组内的结构
          const objSchema = generateJSONSchema(item.children || [], `${parentPath}${item.name}[].`)
          schema.properties[item.name] = {
            type: "array",
            items: {
              type: "object",
              properties: objSchema.properties
            }
          }
          
          if (objSchema.required && objSchema.required.length) {
            schema.properties[item.name].items.required = objSchema.required
          }
        } else {
          // 如果没有子元素，则使用空对象作为数组元素
          schema.properties[item.name] = {
            type: "array",
            items: { type: "object", properties: {} }
          }
        }
      } else {
        schema.properties[item.name] = {
          type: "array",
          items: { type: arrayType === 'integer' ? 'integer' : arrayType }
        }
      }
      
      if (item.description) {
        schema.properties[item.name].description = item.description
      }
      
      if (item.required) {
        schema.required.push(item.name)
      }
    } else if (item.type === 'enum' && item.enumValues) {
      schema.properties[item.name] = { 
        type: "string",
        enum: item.enumValues.split(',').map(v => v.trim())
      }
      
      if (item.description) {
        schema.properties[item.name].description = item.description
      }
      
      if (item.required) {
        schema.required.push(item.name)
      }
    } else {
      schema.properties[item.name] = { 
        type: item.type === 'integer' ? 'integer' : item.type 
      }
      
      if (item.description) {
        schema.properties[item.name].description = item.description
      }
      
      if (item.required) {
        schema.required.push(item.name)
      }
    }
  })
  
  if (schema.required.length === 0) {
    delete schema.required
  }
  
  return schema
}

// 添加新元素
function addItem(parentPath = []) {
  if (!newItemName.value) return
  // 检查名称是否已存在
  const existingItem = jsonStructure.value.find(item => item.name === newItemName.value)
  if (existingItem) {
    alert(`名称 "${newItemName.value}" 已存在，请使用其他名称`)
    newItemName.value = ""
    return
  }
  // 验证名称是否符合要求 字母开头，只能包含字母、数字、下划线
  const namePattern = /^[a-zA-Z][a-zA-Z0-9_]*$/
  if (!namePattern.test(newItemName.value)) {
    alert(`${newItemName.value} 无效, 只能以字母开头，并且只能包含字母、数字和下划线`)
    newItemName.value = ""
    return
  }
  
  const newItem = {
    name: newItemName.value,
    type: newItemType.value,
    required: newItemRequired.value,
    description: newItemDescription.value || undefined
  }
  
  if (newItemType.value === 'object') {
    newItem.children = []
  } else if (newItemType.value === 'array[object]') {
    newItem.children = []
  } else if (newItemType.value === 'enum' || newItemType.value === 'array[enum]') {
    newItem.enumValues = enumValues.value
  }

  // 检查当前选择的项是否允许添加子项 (对象或对象数组)
  const canAddToSelected = currentEditingItem.value && 
    (currentEditingItem.value.type === 'object' || currentEditingItem.value.type === 'array[object]')
  
  if (!parentPath.length || !canAddToSelected) {
    // 添加到根级别
    jsonStructure.value.push(newItem)
  } else {
    // 添加到选中的对象或对象数组中
    let current = jsonStructure.value
    let targetItem = null
    
    for (let i = 0; i < parentPath.length; i++) {
      const index = parentPath[i]
      if (i === parentPath.length - 1) {
        // 找到最后一级，这就是我们要添加子元素的对象
        targetItem = current[index]
      } else {
        // 继续沿着路径向下
        if (!current[index].children) {
          current[index].children = []
        }
        current = current[index].children
      }
    }
    
    // 确保目标项有children数组
    if (targetItem && (targetItem.type === 'object' || targetItem.type === 'array[object]')) {
      if (!targetItem.children) {
        targetItem.children = []
      }
      targetItem.children.push(newItem)
    }
  }
  
  // 重置表单
  newItemName.value = ''
  newItemType.value = 'string'
  newItemRequired.value = false
  newItemDescription.value = ''
  enumValues.value = ''
}

// 选择元素进行编辑
function selectItem(index, path = []) {
  selectedIndex.value = index
  selectedPath.value = [...path, index]
  
  // 获取当前选中的项目引用
  let item = null
  if (path.length === 0) {
    item = jsonStructure.value[index]
  } else {
    let current = jsonStructure.value
    for (let i = 0; i < path.length; i++) {
      const pathIndex = path[i]
      current = current[pathIndex].children || []
      if (i === path.length - 1) {
        item = current[index]
      }
    }
  }
  
  currentEditingItem.value = item
}

// 删除元素
function removeItem(index, parentPath = []) {
  if (parentPath.length === 0) {
    jsonStructure.value.splice(index, 1)
  } else {
    let current = jsonStructure.value
    for (let i = 0; i < parentPath.length; i++) {
      const pathIndex = parentPath[i]
      if (i === parentPath.length - 1) {
        current[pathIndex].children.splice(index, 1)
      } else {
        current = current[pathIndex].children
      }
    }
  }
  
  // 重置选择
  if (JSON.stringify(selectedPath.value) === JSON.stringify([...parentPath, index])) {
    selectedIndex.value = null
    selectedPath.value = []
  }
}

// 获取当前选中的项目的父对象
function getParentOfSelected() {
  if (!selectedPath.value || selectedPath.value.length === 0) {
    return { parent: jsonStructure.value, index: null }
  }
  
  const path = [...selectedPath.value]
  const currentIndex = path.pop()
  
  let parent = jsonStructure.value
  let current = parent
  
  for (let i = 0; i < path.length; i++) {
    const index = path[i]
    current = current[index].children
    if (i === path.length - 1) {
      parent = current
    }
  }
  
  return { parent, index: currentIndex }
}

// 可以添加子节点的项目类型
function canAddChildren(item) {
  return item && (item.type === 'object' || item.type === 'array[object]')
}

// 获取当前编辑路径的显示名称
function getEditingPathName() {
  if (!selectedPath.value || selectedPath.value.length === 0) {
    return null
  }
  
  let names = []
  let current = jsonStructure.value
  
  for (let i = 0; i < selectedPath.value.length; i++) {
    const index = selectedPath.value[i]
    if (i === selectedPath.value.length - 1) {
      names.push(current[index].name)
    } else {
      names.push(current[index].name)
      current = current[index].children
    }
  }
  
  return names.join(' > ')
}

// 获取深度对应的类样式
function getIndentClass(depth) {
  return `indent-level-${depth}`
}

// 获取元素的嵌套深度类
function getDepthClass(depth) {
  return `depth-${depth}`
}
</script>

<template>
  <div class="container">
    <!-- <p class="description">定义JSON结构，自动生成OpenAI structured output的JSON Schema</p> -->
    
    <div class="app-container">
      <!-- 左侧：JSON结构定义 -->
      <div class="structure-panel">
        <div class="form-group">
          <div class="form-row">
            <label>名称:</label>
            <input type="text" v-model="newItemName" placeholder="属性名称" />
          </div>
          <div class="form-row">
            <label>类型:</label>
            <select v-model="newItemType">
              <option value="string">字符串 (string)</option>
              <option value="number">数字 (number)</option>
              <option value="integer">整数 (integer)</option>
              <option value="boolean">布尔值 (boolean)</option>
              <option value="enum">枚举 (enum)</option>
              <option value="object">对象 (object)</option>
              <option value="array[string]">字符串数组 (array[string])</option>
              <option value="array[number]">数字数组 (array[number])</option>
              <option value="array[integer]">整数数组 (array[integer])</option>
              <option value="array[boolean]">布尔值数组 (array[boolean])</option>
              <option value="array[enum]">枚举数组 (array[enum])</option>
              <option value="array[object]">对象数组 (array[object])</option>
            </select>
          </div>
          <div class="form-row" v-if="newItemType === 'enum' || newItemType === 'array[enum]'">
            <label>枚举值:</label>
            <input type="text" v-model="enumValues" placeholder="逗号分隔的枚举值" />
          </div>
          <div class="form-row">
            <label>必填:</label>
            <input type="checkbox" v-model="newItemRequired" />
          </div>
          <div class="form-row">
            <label>描述:</label>
            <input type="text" v-model="newItemDescription" placeholder="可选描述" />
          </div>
          <!-- 当前选择状态 -->
        <div v-if="currentEditingItem" class="current-selection">
          <div class="selected-info">
            <strong>当前选中:</strong> 
            <span class="selected-path">{{ getEditingPathName() }}</span>
            <span class="selected-type">({{ currentEditingItem.type }})</span>
            <button class="reset-selection-btn" @click="selectedPath = []; currentEditingItem = null;">
              返回根级
            </button>
          </div>
        </div>
          <button class="add-btn" @click="addItem(selectedPath)">
              {{ currentEditingItem && (currentEditingItem.type === 'object' || currentEditingItem.type === 'array[object]') 
              ? '添加子元素到 ' + currentEditingItem.name
              : '添加根级元素' }}
          </button>
          
          <!-- 提示信息 -->
          <div v-if="currentEditingItem && !(currentEditingItem.type === 'object' || currentEditingItem.type === 'array[object]')" class="info-message">
            注意: 只能向对象(object)或对象数组(array[object])添加子元素
          </div>
        </div>
        
        <!-- 结构树 -->
        <div class="structure-tree">
          <TreeItem 
            v-for="(item, index) in jsonStructure" 
            :key="`root-${index}`"
            :item="item" 
            :index="index" 
            :path="[]" 
            :selectedPath="selectedPath"
            @select="selectItem"
            @remove="removeItem"
          />
        </div>
      </div>
      
      <!-- 右侧：Tabs 显示 -->
      <div class="output-panel">
        <div class="tabs">
          <div 
            class="tab" 
            :class="{ active: activeTab === 'zod' }"
            @click="activeTab = 'zod'"
          >
            JSON Schema
          </div>
          <div 
            class="tab" 
            :class="{ active: activeTab === 'openai' }"
            @click="activeTab = 'openai'"
          >
            OpenAI Example
          </div>
        </div>
        
        <div class="tab-content">
          <!-- Zod Schema Tab -->
          <div v-if="activeTab === 'zod'" class="schema-output">
            <JsonViewer :data="zodOpenaiSchema" />
          </div>
          
          <!-- OpenAI 使用示例 Tab -->
          <div v-if="activeTab === 'openai'" class="code-wrapper">
            <div class="code-header">
              <span>使用方法示例</span>
            </div>
            <pre class="code-sample">
const json_schema = {{JSON.stringify(openaiSchema, null, 2)}}
const response = await openai.beta.chat.completions.parse({
  model: "gpt-4o-mini",
  messages: [
    { role: "system", content: "You are a helpful assistant." },
    { role: "user", content: "Generate a mock data." }
  ],
  response_format: {
    type: "json_schema", 
    json_schema
  }
});</pre>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style>
body {
  font-family: 'Arial', sans-serif;
  margin: 0;
  padding: 0;
  background-color: #f5f5f5;
  color: #333;
}

.container {
  /* width: 95%; */
  margin: 0 auto;
  padding: 10px;
}

h1 {
  text-align: center;
  color: #2c3e50;
}

.description {
  text-align: center;
  color: #666;
  margin-bottom: 30px;
}

.app-container {
  display: flex;
  min-height: 500px;
  justify-content: space-around;
  height: calc(100vh - 20px);
}

.structure-panel, .output-panel {
  min-width: 46%;
  flex: none;
  background: #fff;
  border-radius: 8px;
  padding: 20px;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
  display: flex;
  flex-direction: column;
  overflow: hidden;
}

h2 {
  margin-top: 0;
  padding-bottom: 10px;
  border-bottom: 1px solid #eee;
  color: #2c3e50;
}

/* 表单样式 */
.form-group {
  margin-bottom: 20px;
  padding: 15px;
  background: #f9f9f9;
  border-radius: 5px;
}

.form-row {
  display: flex;
  align-items: center;
  margin-bottom: 10px;
}

.form-row label {
  width: 80px;
  font-weight: bold;
}

.form-row input[type="text"],
.form-row select {
  flex: 1;
  padding: 8px;
  border: 1px solid #ddd;
  border-radius: 4px;
}

.add-btn {
  background: #4caf50;
  color: white;
  border: none;
  padding: 8px 16px;
  border-radius: 4px;
  cursor: pointer;
  font-weight: bold;
}

.add-btn:hover {
  background: #45a049;
}

/* 当前选择状态样式 */
.current-selection {
  background: #ecf5ff;
  padding: 10px;
  border-radius: 5px;
  margin-bottom: 15px;
  border-left: 4px solid #409eff;
}

.selected-info {
  display: flex;
  align-items: center;
  gap: 5px;
}

.selected-path {
  font-weight: 500;
  color: #409eff;
}

.selected-type {
  color: #666;
  font-size: 0.9em;
}

.form-title {
  color: #2c3e50;
  margin-bottom: 15px;
  padding-bottom: 5px;
  border-bottom: 1px dashed #ddd;
}

.info-message {
  margin-top: 10px;
  padding: 8px;
  background: #fff3cd;
  border-left: 3px solid #ffc107;
  font-size: 0.9em;
  color: #856404;
}

/* 结构树样式 */
.structure-tree {
  flex: 1;
  overflow-y: auto;
  max-height: 400px;
}

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
}

.child-item {
  margin-top: 5px;
}

.grandchild-item {
  margin-top: 5px;
}

/* Tab 样式 */
.tabs {
  display: flex;
  border-bottom: 1px solid #ddd;
  margin-bottom: 15px;
}

.tab {
  padding: 10px 20px;
  cursor: pointer;
  border-bottom: 2px solid transparent;
  font-weight: 500;
  color: #666;
  transition: all 0.2s ease;
}

.tab:hover {
  color: #409eff;
}

.tab.active {
  color: #409eff;
  border-bottom-color: #409eff;
}

.tab-content {
  flex: 1;
  overflow: auto;
  background: #f9f9f9;
  border-radius: 5px;
}

.schema-output {
  height: 100%;
  background: #272822;
  padding: 15px;
  border-radius: 5px;
  overflow: auto;
}

.schema-output pre {
  margin: 0;
  white-space: pre-wrap;
  font-family: 'Monaco', 'Menlo', 'Ubuntu Mono', 'Consolas', 'source-code-pro', monospace;
  font-size: 14px;
}

.schema-usage {
  height: 100%;
}

.code-wrapper {
  height: 100%;
  display: flex;
  flex-direction: column;
  background: #272822;
  color: #f8f8f2;
  border-radius: 5px;
  overflow: auto;
  text-align: left;
}

.code-header {
  padding: 8px 15px;
  background: #1d1e19;
  border-bottom: 1px solid #3e3d32;
  font-family: sans-serif;
  font-size: 0.9em;
}

.code-sample {
  margin: 0;
  padding: 15px;
  white-space: pre-wrap;
  font-family: 'Monaco', 'Menlo', 'Ubuntu Mono', 'Consolas', 'source-code-pro', monospace;
  font-size: 14px;
  line-height: 1.5;
  overflow-x: auto;
}

.schema-in-code {
  padding: 0 15px;
  margin-left: 20px;
  border-left: 3px solid #444;
}

.indent-level-0 { margin-left: 0; }
.indent-level-1 { margin-left: 20px; }
.indent-level-2 { margin-left: 40px; }
.indent-level-3 { margin-left: 60px; }
.indent-level-4 { margin-left: 80px; }
</style>
