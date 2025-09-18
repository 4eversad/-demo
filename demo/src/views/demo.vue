<template>
  <div class="page">
    <div class="left">
      <div class="toolbar">
        <button class="btn" @click="buildGraph">从文本生成流程图</button>
        <button class="btn" @click="exportText">从图生成文本</button>
        <span class="hint">示例：登录 -> 验证短信 -> 进入主页</span>
      </div>
      <textarea
        v-model="inputText"
        class="editor"
        placeholder="在此粘贴或输入流程文本，支持两种语法：\n1) 简写：登录 -> 验证短信 -> 进入主页\n2) DSL：\n1.开始\n2.输入数据\n3.处理\n4.展示\n5.结束\n1->2->3\n2->|连线|4->5\n3->5"
      />
    </div>
    <div class="right">
      <div ref="containerRef" class="graph" />
    </div>
  </div>
</template>

<script setup>
import { onMounted, onBeforeUnmount, ref } from 'vue'
import { Graph } from '@antv/x6'

const containerRef = ref(null)
let graph = null

const inputText = ref(
  '1.开始1\n2.输入数据33\n3.数据处理\n数据清洗、数据校对\n4.数据展示\n5.结束\n1->2->3\n2->|连线|4->5\n3->5'
)

const nodeIdToLabel = new Map()
const labelToNodeId = new Map()

function resetMaps() {
  nodeIdToLabel.clear()
  labelToNodeId.clear()
}

function parseSimpleChain(line) {
  // 形如：A -> B -> C
  const parts = line
    .split(/-+>|→/)
    .map((s) => s.trim())
    .filter(Boolean)
  if (parts.length <= 1) return null
  const nodes = Array.from(new Set(parts))
  const edges = []
  for (let i = 0; i < parts.length - 1; i++) {
    edges.push([parts[i], parts[i + 1], ''])
  }
  return { nodes, edges }
}

function parseDsl(text) {
  // DSL 由两部分构成：编号节点定义 与 连接定义
  // 节点定义：`1.开始` 或多行描述：上一行没有编号，视为上一编号节点的描述续行
  // 连接定义：`1->2->3`，支持带标签：`2->|连线|4->5`
  const lines = text.split(/\r?\n/)
  const idToLabel = new Map()
  let lastId = null
  const edgeLines = []

  for (const raw of lines) {
    const line = raw.trim()
    if (!line) continue
    const m = line.match(/^(\d+)\.(.+)$/)
    if (m) {
      lastId = m[1]
      idToLabel.set(lastId, m[2].trim())
      continue
    }
    // 如果是续行描述
    if (lastId && !/->/.test(line)) {
      const prev = idToLabel.get(lastId) || ''
      idToLabel.set(lastId, (prev ? prev + '\n' : '') + line)
      continue
    }
    if (/->/.test(line)) edgeLines.push(line)
  }

  const nodes = []
  const edges = []
  for (const [id, label] of idToLabel.entries()) {
    nodes.push({ id, label })
  }

  for (const el of edgeLines) {
    // 解析 2->|连线|4->5 或 1->2->3
    const segs = el.split('->')
    const hops = []
    for (const s of segs) {
      const mm = s.match(/^\|(.*)\|(.+)$/)
      if (mm) {
        hops.push({ label: mm[1].trim(), target: mm[2].trim() })
      } else {
        hops.push({ label: '', target: s.trim() })
      }
    }
    for (let i = 0; i < hops.length - 1; i++) {
      const from = hops[i].target
      const to = hops[i + 1].target
      const lbl = hops[i + 1].label || ''
      edges.push([from, to, lbl])
    }
  }

  return { nodes, edges, idToLabel }
}

function buildGraphFromParsed(parsed) {
  const g = graph
  if (!g) return
  g.clearCells()
  resetMaps()

  // 生成节点
  const added = new Map()
  const toText = (val) => (val == null ? '' : String(val))
  const getStableId = (label) => 'n_' + encodeURIComponent(toText(label).trim())
  const ensureNode = (label) => {
    const key = toText(label).trim()
    if (added.has(key)) return added.get(key)
    const id = getStableId(key)
    added.set(key, id)
    nodeIdToLabel.set(id, key)
    labelToNodeId.set(key, id)
    return id
  }

  let nodes = []
  let edges = []

  const isSimple =
    Array.isArray(parsed.nodes) &&
    parsed.nodes.length > 0 &&
    typeof parsed.nodes[0] === 'string'
  if (isSimple) {
    // 简写：节点用基于标签的稳定 ID，保证与边一致
    nodes = parsed.nodes.map((label) => ({ id: ensureNode(label), label }))
    edges = parsed.edges.map(([s, t, lbl]) => ({
      source: ensureNode(s),
      target: ensureNode(t),
      label: lbl
    }))
  } else {
    // DSL
    const definedIdSet = new Set()
    for (const n of parsed.nodes) {
      const id = 'n_' + n.id
      definedIdSet.add(String(n.id))
      nodeIdToLabel.set(id, n.label)
      labelToNodeId.set(n.label, id)
      nodes.push({ id, label: n.label })
    }
    // 先解析边
    edges = parsed.edges.map(([s, t, lbl]) => ({
      source: 'n_' + s,
      target: 'n_' + t,
      label: lbl
    }))
    // 自动补齐未定义但被边引用的节点（占位标签使用其编号）
    const referenced = new Set()
    for (const [s, t] of parsed.edges) {
      referenced.add(String(s))
      referenced.add(String(t))
    }
    for (const idNum of referenced) {
      if (!definedIdSet.has(idNum)) {
        const nid = 'n_' + idNum
        nodes.push({ id: nid, label: String(idNum) })
        nodeIdToLabel.set(nid, String(idNum))
        labelToNodeId.set(String(idNum), nid)
      }
    }
  }

  // 统一兜底：若还有边引用了不存在的节点，按边反推补齐
  const nodeIdSet = new Set(nodes.map((n) => n.id))
  const decodeLabelFromId = (id) => {
    if (id.startsWith('n_')) {
      const raw = id.slice(2)
      try {
        return decodeURIComponent(raw)
      } catch (e) {
        return raw
      }
    }
    return id
  }
  for (const e of edges) {
    if (!nodeIdSet.has(e.source)) {
      const lbl = decodeLabelFromId(e.source)
      nodes.push({ id: e.source, label: lbl })
      nodeIdToLabel.set(e.source, lbl)
      labelToNodeId.set(lbl, e.source)
      nodeIdSet.add(e.source)
    }
    if (!nodeIdSet.has(e.target)) {
      const lbl = decodeLabelFromId(e.target)
      nodes.push({ id: e.target, label: lbl })
      nodeIdToLabel.set(e.target, lbl)
      labelToNodeId.set(lbl, e.target)
      nodeIdSet.add(e.target)
    }
  }

  const width = 160
  const height = 48
  const gapX = 60
  const gapY = 40

  // 简易网格布局：多行排列
  const maxPerRow = Math.ceil(Math.sqrt(nodes.length || 1))
  nodes.forEach((n, idx) => {
    const row = Math.floor(idx / maxPerRow)
    const col = idx % maxPerRow
    n.x = 40 + col * (width + gapX)
    n.y = 40 + row * (height + gapY)
  })

  g.addNodes(
    nodes.map((n) => ({
      id: n.id,
      x: n.x,
      y: n.y,
      width,
      height,
      shape: 'rect',
      attrs: {
        body: { stroke: '#5F95FF', fill: '#F7F9FF', rx: 6, ry: 6 },
        label: { text: n.label, fill: '#1D2129', fontSize: 14 }
      }
    }))
  )

  g.addEdges(
    edges.map((e) => ({
      source: e.source,
      target: e.target,
      attrs: {
        line: { stroke: '#A9AEB8', targetMarker: 'classic', strokeWidth: 1.4 }
      },
      labels: e.label
        ? [
            {
              position: 0.5,
              attrs: {
                label: { text: e.label, fill: '#6B7785' },
                body: { fill: '#fff' }
              }
            }
          ]
        : []
    }))
  )
}

function tryBuildFromText() {
  const text = inputText.value.trim()
  if (!text) {
    graph?.clearCells()
    resetMaps()
    return
  }
  // 优先尝试 DSL，若没解析到节点且形如单行箭头链，则走简写
  const dsl = parseDsl(text)
  if (dsl.nodes.length > 0) {
    buildGraphFromParsed(dsl)
    return
  }
  const simple = parseSimpleChain(text)
  if (simple) buildGraphFromParsed(simple)
}

function buildGraph() {
  tryBuildFromText()
}

function exportText() {
  if (!graph) return
  // 如果是 DSL 构造（节点 id 以 n_数字 命名），导出 DSL；否则导出箭头链
  const cells = graph.getCells()
  const nodes = cells.filter((c) => c.isNode())
  const edges = cells.filter((c) => c.isEdge())

  const isDsl = nodes.every((n) => /^n_\d+$/.test(n.id))
  if (isDsl) {
    // 导出编号 + 连接
    const list = []
    const idMap = new Map()
    let idx = 1
    for (const n of nodes) {
      const original = n.id.replace(/^n_/, '')
      idMap.set(n.id, original)
      const label = n.attr('label/text') || nodeIdToLabel.get(n.id) || ''
      list.push(`${original}.${label}`)
    }
    // 简单保持顺序不变输出
    for (const e of edges) {
      const src = idMap.get(e.getSourceCellId())
      const tgt = idMap.get(e.getTargetCellId())
      const lbl = e.getLabels()?.[0]?.attrs?.label?.text || ''
      if (lbl) {
        list.push(`${src}->|${lbl}|${tgt}`)
      } else {
        list.push(`${src}->${tgt}`)
      }
    }
    inputText.value = list.join('\n')
    return
  }

  // 否则导出为 A->B->C 的链条（可能非单链，退化为多行）
  const label = (id) =>
    graph.getCellById(id)?.attr('label/text') || nodeIdToLabel.get(id) || ''
  const lines = []
  for (const e of edges) {
    const src = label(e.getSourceCellId())
    const tgt = label(e.getTargetCellId())
    const lbl = e.getLabels()?.[0]?.attrs?.label?.text || ''
    if (lbl) lines.push(`${src} -> |${lbl}| ${tgt}`)
    else lines.push(`${src} -> ${tgt}`)
  }
  inputText.value = lines.join('\n')
}

function bindGraphEvents() {
  // 双击编辑节点文字，并同步回文本
  graph.on('node:dblclick', ({ node }) => {
    const old = node.attr('label/text') || ''
    const next = window.prompt('编辑节点文本', old)
    if (next == null) return
    node.attr('label/text', next)
    const id = node.id
    const prev = nodeIdToLabel.get(id)
    if (prev) labelToNodeId.delete(prev)
    nodeIdToLabel.set(id, next)
    labelToNodeId.set(next, id)
    exportText()
  })

  // 双击边标签
  graph.on('edge:dblclick', ({ edge }) => {
    const cur = edge.getLabels()?.[0]?.attrs?.label?.text || ''
    const next = window.prompt('编辑连线标签', cur)
    if (next == null) return
    if (next) {
      edge.setLabels([
        {
          position: 0.5,
          attrs: { label: { text: next }, body: { fill: '#fff' } }
        }
      ])
    } else {
      edge.setLabels([])
    }
    exportText()
  })
}

onMounted(() => {
  graph = new Graph({
    container: containerRef.value,
    grid: true,
    panning: true,
    mousewheel: { enabled: true, modifiers: ['ctrl', 'meta'] },
    connecting: {
      router: 'manhattan',
      connector: 'rounded',
      allowBlank: false,
      snap: true,
      validateConnection({ targetMagnet }) {
        return !!targetMagnet
      },
      createEdge() {
        return graph.createEdge({
          attrs: {
            line: {
              stroke: '#A9AEB8',
              targetMarker: 'classic',
              strokeWidth: 1.4
            }
          }
        })
      }
    }
  })
  bindGraphEvents()
  tryBuildFromText()
})

onBeforeUnmount(() => {
  graph?.dispose()
  graph = null
})
</script>

<style lang="scss" scoped>
.page {
  display: flex;
  height: calc(100vh - 20px);
  padding: 10px;
  box-sizing: border-box;
  gap: 10px;
}
.left {
  width: 420px;
  display: flex;
  flex-direction: column;
  gap: 8px;
}
.right {
  flex: 1;
  min-width: 0;
}
.toolbar {
  display: flex;
  align-items: center;
  gap: 8px;
}
.btn {
  background: #1677ff;
  color: #fff;
  border: none;
  padding: 6px 10px;
  border-radius: 4px;
  cursor: pointer;
}
.btn:hover {
  background: #3c89ff;
}
.hint {
  color: #86909c;
  font-size: 12px;
}
.editor {
  flex: 1;
  width: 100%;
  resize: none;
  font-family: ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas,
    'Liberation Mono', 'Courier New', monospace;
  font-size: 13px;
  line-height: 1.6;
  padding: 10px;
  border: 1px solid #e5e6eb;
  border-radius: 6px;
}
.graph {
  width: 100%;
  height: 100%;
  border: 1px solid #e5e6eb;
  border-radius: 6px;
  background: #fff;
}
</style>
