<template>
  <div class="container">
    <header>
      <h1>文件压缩工具</h1>
      <p class="subtitle">上传文件或文件夹，保留原始目录结构创建 ZIP 压缩文件</p>
    </header>

    <div class="main-content">
      <div class="upload-section">
        <h2 class="section-title">上传文件/文件夹</h2>
        <div ref="dropZone" class="drop-area" @dragover.prevent @drop.prevent="handleDropFiles">
          <i>📁</i>
          <p>拖放文件或文件夹到此处</p>
          <p>或</p>
          <el-tooltip
            effect="dark"
            placement="top"
          >
            <button ref="browseBtn" class="btn">操作</button>
            <template #content>
              <div class="tooltip-content">
                <div @click="fileInput.click()">选择文件</div>
                <div @click="fileFolderInput.click()">选择文件夹</div>
              </div>
            </template>
          </el-tooltip>
          <input ref="fileFolderInput" type="file" class="file-input" multiple webkitdirectory @change="(e) =>handleFiles(e.target.files)" />
          <input ref="fileInput" type="file" class="file-input" multiple @change="(e) =>handleFiles(e.target.files)" />
        </div>
      </div>

      <div class="preview-section">
        <h2 class="section-title">目录结构预览</h2>
        <div class="file-tree" id="fileTree">
          <el-tree-v2
            :data="treeData"
            :height="250"
            empty-text="暂无文件"
          >
            <template #default="{ node, data }">
              <div class="file-tree-item">
                <el-icon v-if="!node.isLeaf"><Folder /></el-icon>
                <span>{{ node.label }}</span>
              </div>
            </template>
          </el-tree-v2>
        </div>

        <div class="status-bar">
          <div class="file-count">文件数: <span>{{ fileCount }}</span></div>
          <div class="folder-count">文件夹数: <span>{{ folderCount }}</span></div>
        </div>
      </div>
    </div>

    <div class="actions">
      <button class="btn btn-download" :disabled="!fileCount" @click="downloadZip">下载 ZIP 文件</button>
      <button class="btn btn-clear" @click="clearAll">清除所有</button>
    </div>

    <footer>
      <p>保留完整目录层级结构 | 支持文件夹上传</p>
    </footer>
  </div>
</template>

<script setup>
import JSZip from 'jszip'
import { nanoid } from 'nanoid'
import { ref, watch, computed } from 'vue'
import { ElMessageBox , ElMessage } from 'element-plus'

const dropZone = ref(null)
const browseBtn = ref(null)
const fileInput = ref(null)
const fileFolderInput = ref(null)

const fileCount = ref(0)
const folderCount = ref(0)

const treeData = ref([])

let files = []

function handleDropFiles(e) {
  if (e.dataTransfer.items) {
    // 处理文件夹拖放
    const items = e.dataTransfer.items;
    for (let i = 0; i < items.length; i++) {
      if (items[i].kind === 'file') {
        const entry = items[i].webkitGetAsEntry();
        if (entry) {
          traverseFileTree(entry);
        }
      }
    }
  } else {
    // 处理文件拖放
    handleFiles(e.dataTransfer.files);
  }
}

// 遍历文件树（处理文件夹）
function traverseFileTree(entry, path = '') {
  if (entry.isFile) {
    entry.file(file => {
      files.push({
        file: file,
        path: path + file.name
      });
      updateFileTree();
    });
  } else if (entry.isDirectory) {
    const dirReader = entry.createReader();
    dirReader.readEntries(entries => {
      for (let i = 0; i < entries.length; i++) {
        traverseFileTree(entries[i], path + entry.name + '/');
      }
    });
  }
}

// 处理文件列表
function handleFiles(fileList) {
  for (let i = 0; i < fileList.length; i++) {
    files.push({
      file: fileList[i],
      path: fileList[i].webkitRelativePath || fileList[i].name
    })
  }
  updateFileTree()
}

function updateFileTree() { 
  if (files.length === 0 ) {
    return
  }
  fileCount.value = 0
  folderCount.value = 0

  treeData.value = convertPathsToTree(files)
}

function convertPathsToTree (paths) {
  // 创建根节点
  const root = { label: 'root', path: '', children: [] };
  
  paths.forEach(item => {
    const parts = item.path.split('/');
    let currentLevel = root;
    let currentPath = '';
    
    parts.forEach((part, index) => {
      const isFile = index === parts.length - 1;
      currentPath = currentPath ? `${currentPath}/${part}` : part;
      
      // 检查节点是否存在
      let node = currentLevel.children?.find(child => child.label === part);
      
      // 创建新节点
      if (!node) {
        node = {
          id: nanoid(),
          label: part,
          path: currentPath,
          isLeaf: isFile,
          file: isFile ? item.file : null
        };
        
        if (!isFile) {
          node.children = [];
          folderCount.value++;
        } else {
          fileCount.value++;
        }
        
        currentLevel.children.push(node);
      }
      
      currentLevel = node;
    });
  });
  
  return root.children || [];
};
function downloadZip() {
  if (files.length === 0) {
    return;
  }

  ElMessageBox.prompt('请输入压缩文件名称', '提示', {
    confirmButtonText: '确定',
    cancelButtonText: '取消',
    inputValidator: (value) => {
      if (!value) {
        return '压缩文件名称不能为空';
      }
      return true;
    },
    inputValue: 'zip-file',
  }).then(({ value }) => {
    const zip = new JSZip();
    files.forEach(item => {
      zip.file(item.path, item.file);
    });

    zip.generateAsync({ type: 'blob' }).then(content => {
      const url = URL.createObjectURL(content);
      const a = document.createElement('a');
      a.href = url;
      a.download = `${value}.zip`;
      document.body.appendChild(a);
      a.click();
      document.body.removeChild(a);
      URL.revokeObjectURL(url);
    });
  }).catch(() => {})
}
function clearAll() {
  files = [];
  treeData.value = [];
  fileCount.value = 0;
  folderCount.value = 0;
}
</script>