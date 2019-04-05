# VueDynamicForm
基于Vue Element 的动态渲染表单组件
  在xxx信息管理这种业务场景中我认为最常见的操作就是对字段的处理(例如查询、编辑等区域的表单、图表的列名、表格的列名)，而字段恰恰是最为 '规范的'，它有自己的名称、类型
> 例如 
> 1. name 它代表名称，类型为字符串，在页面中应该是一个文本框
> 2. sex 它代表性别，类型为数值型，当它为0的时候代表男、为1的时候代表女,在页面中应该是一个下拉框


##### 我们可以通过程序语言来描述这种数据结构
用对象或者说map这种结构而不用数组是为了可以精准对某个字段进行设置 而数组需要先遍历查找到这个字段再进行设置
```
let fieldMap = {
  name: {
    name: 'name',
    label: '名称',
    type: 'text'
  },
  sex: {
    name: 'sex',
    label: '性别',
    type: 'select',
    options: {
      list: [
        {
          label: '男', value: 0
        },
        {
          label: '女', value: 1
        }
      ]
    }
  }
}
```
##### 我们可以轻易的把这种数据结构渲染成表单
可通过 formData 这个外部传入的对象来对数据进行统一的设置与读取
```
 <!-- 动态表单的使用  -->
 <dynamic-form :field-map="fieldMap" :form-data="formData" />
```
##### 动态表单的简易实现
```
 <!-- 动态表单的内部实现  -->
<section>
    <el-form  :disabled='disabled' :inline="inline">
      <template v-for="item in fieldMap">
        <el-form-item :prop="item.name" :label="item.label" :key="item.name">
          <!-- 文本框 -->
          <template v-if="item.type ==='text'">
            <el-input v-model="formData[item.name]" type="text" :placeholder="item.placeholder" />
          </template>
          <!-- 下拉选择框 -->
          <template v-else-if="item.type==='select'">
            <el-select v-model="formData[item.name]" :placeholder="item.placeholder">
              <template v-for="optionItem in item.options.list">
                <el-option :label="optionItem.label" :value="optionItem.value" :key="optionItem.value"/>
              </template>
            </el-select>
          </template>
        </el-form-item>
      </template>
    </el-form>
  </section>
```
```
export default {
  name: 'dynamicForm',
  props: {
    fieldMap: {
      type: Object,
      default: () => {
        return {}
      }
    },
    formData: {
      type: Object,
      default: () => {
        return {}
      }
    },
    disabled: {
      type: Boolean,
      default: false
    },
    inline: {
      type: Boolean,
      default: true
    }
  }
}
```
##### 同一个字段应有不同的使用(显示、禁用)场景
例如有的字段可以查询但不能编辑，我们可以引入一个场景的概念就可以轻易解决这个问题
```
let fieldMap = {
  name: {
    name: 'name',
    label: '名称',
    useScene: 'list||update||query',
    type: 'text'
  },
  sex: {
    name: 'sex',
    label: '性别',
    type: 'select',
    useScene: 'update||query',
    options: {
      list: [
        {
          label: '男', value: 0
        },
        {
          label: '女', value: 1
        }
      ]
    }
  }
}
```
名称既可以显示又可以编辑与查询而性别只能编辑与查询但不能显示，useScene是使用场景如果字段不支持这个场景那么字段不予显示(可自行实现禁用场景)
```
 <!-- 查询区域  -->
 <dynamic-form :field-map="fieldMap" :form-data="formData" scene="query"/>
 <!-- 编辑区域 -->
 <dynamic-form :field-map="fieldMap" :form-data="formData" scene="update"/>
```
##### 场景的实现
```
 <!-- 动态表单的内部实现  -->
<section>
    <el-form  :disabled='disabled' :inline="inline">
      <template v-for="item in fieldMap">
        <el-form-item :prop="item.name" :label="item.label" :key="item.name" v-if="m_canUse(item)">
          <!-- 文本框 -->
          <template v-if="item.type ==='text'">
            <el-input v-model="formData[item.name]" type="text" :placeholder="item.placeholder" />
          </template>
        </el-form-item>
      </template>
    </el-form>
  </section>
```
```
export default {
  name: 'dynamicForm',
  props: {
    scene: {
      type: String,
      default: '*'
     },
     sceneMap:{
      type: Object,
       default: () => {
         return {}
       }
      }
  },
  methods: {
    m_canUse (field, scene, sceneMap, sceneParser) {
      scene = scene || this.scene
      sceneMap = sceneMap || this.sceneMap || {}
      sceneParser = sceneParser || this.m_sceneParser
      if (check.isBoolean(field.canUse)) {
        return field.canUse
      } else if (!field.useScene) {
        return false
      } else {
        return sceneParser(scene, field.useScene, sceneMap)
      }
    },
    m_sceneParser (scene, useScene, sceneMap) {
      if (!useScene) {
        return false
      }
      const symbolList = ['(', ')', '&', '|', '!']
      let evalStr = ''
      let word = ''
      for (let c of useScene) {
        if (~symbolList.indexOf(c)) {
          if (word !== '') {
            evalStr += `${scene === word || !!sceneMap[word]}`
            word = ''
          }
          evalStr += c
        } else {
          word += c
        }
      }
      if (word) {
        evalStr += `${scene === word || !!sceneMap[word]}`
      }
      return eval(evalStr)
    }
  }
}
```
重点就在于m_canUse的实现，它用eval取巧的实现了一个场景逻辑字符串转布尔值的一个骚操作
##### 动态场景的实现
看到这里可能有的朋友会很不解，为什么我要构造一个如此复杂的useScene,直接定义 canUpdate canQuery 这种布尔值变量来指定场景不就行了吗?
实际上需求是非常复杂多变的，场景可以说是无限的甚至是相互交织关联的、我们可能会根据用户的操作动态显示字段的显示隐藏，例如提交后显示提交人、提交时间等字段、撤销了就不予显示

```
let fieldMap = {
  submitter: {
    name: 'name',
    label: '提交人',
    useScene: '(list||update||query)&&isSubmitted',
    type: 'text'
  }
}
```
```
<dynamic-form 
    :field-map="fieldMap" 
    :form-data="formData" 
    scene="update"
    :scene-map="sceneMap" />

data () {
 return {
   sceneMap: {
     isSubmitted: false
    }
  }
}
```
我们可以在用户做某些操作时来动态设置sceneMap的状态来达到控制表单的显示、隐藏、禁用，当状态越复杂时你就越能感觉到它的威力

##### 响应表单的事件
可以在动态表单内部监听表单的事件(可查阅相关UI库文档)、当表单事件触发时对外传递事件(携带当前操作的字段信息、$event信息或arguments)
##### 自定义UI到表单的任意位置
有时我们想在任意两个字段之间插入一个非通用的ui组件，我们可以通过具名插槽来实现
```
let fieldMap = {
  name: {
    name: 'name',
    label: '姓名',
    useScene: 'update',
    type: 'text'
  },
 interesting:{
    label:'个性的ui组件',
    name:'interesting',
    isSlot:true
 },
  sexName: {
    name: 'name',
    label: '姓别',
    useScene: 'update',
    type: 'text'
  }
}
```

```
 <dynamic-form 
    :fieldMap="fieldMap" 
    :sceneMap="sceneMap" 
    :form-data="formData" 
    :scene="update">
    <!-- 排序相关插槽 -->
    <template slot="interesting">
       <sb-ui v-model="formData['interesting']" />
    </template>
 </dynamic-form>
```
```
 <!-- 动态表单的内部实现  -->
<section>
    <el-form  :disabled='disabled' :inline="inline">
      <template v-for="item in fieldMap">
       <slot :name="item.name" v-if="item.isSlot"></slot>
        <el-form-item :prop="item.name" :label="item.label" :key="item.name" v-else-if="m_canUse(item)">
          <!-- 文本框 -->
          <template v-if="item.type ==='text'">
            <el-input v-model="formData[item.name]" type="text" :placeholder="item.placeholder" />
          </template>
        </el-form-item>
      </template>
    </el-form>
  </section>
```

这是一个动态表单的简易实现，需要大家结合自身的业务场景去填充各种各样的表单和相关的参数、事件
