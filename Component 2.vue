<script setup lang="ts">
import { type FileMetadata, type FileType } from "importPath";
import {
  preview_icon,
  download_icon,
  remove_icon,
  folder_icon,
  rename_icon,
  accept_icon,
  cancel_icon,
} from "importPath";
import { nextTick, ref, type Ref } from "importPath";
import {
  cutSuffix,
  formatBytes,
  formatDate,
  pathToString,
  SRPToPath,
} from "importPath";
import FileIcons from "importPath";
import FilePreview from "importPath";
import {
  deleteFile,
  downloadFile,
  deleteFolder,
  renameFile,
  renameFolder,
} from "importPath";
import WarningPopup from "importPath";
import { refreshKey, showLoader, userConfig } from "importPath";

const props = defineProps<{
  fileName: string;
  created: string;
  modified: string;
  fileType: FileType;
  dir_path?: string[];
  uri?: string;
  resourcePath: string;
  size?: string;
  doubleView: boolean;
}>();

const emit = defineEmits<{ (e: "update_path", new_path: string[]): void }>();
const curr_dir_path = ref(props.dir_path);
const newTab = ref(false);
const createdFormatted = ref(
  formatDate(new Date(props.created), userConfig.value?.DateTimeFormat!)
);
const modifiedFormatted = ref(
  formatDate(new Date(props.modified), userConfig.value?.DateTimeFormat!)
);

function update_path() {
  if (curr_dir_path.value) {
    curr_dir_path.value = SRPToPath(props.resourcePath);
    emit("update_path", curr_dir_path.value);
  }
}

async function handleChangePath() {
  if(props.fileType !== "folder") {
    handlePreview(true)
  }
  if (props.fileType !== "folder" || (props.fileType === "folder" && show_rename.value)) return;
  update_path();
}

const previewWindow = ref(false);
function handlePreview(auto=false) {
  if (!userConfig.value?.permissions.Preview) return;
  
  if(show_rename.value) {
    return;
  }

  newTab.value = auto
  previewWindow.value = true;
}

const downloading = ref(false);
async function handleDownload() {
  downloading.value = true;
  if (!props.uri || !props.resourcePath) return -1;
  const blob = await downloadFile(props.resourcePath);
  if (!blob) return -1;
  const link = document.createElement("a");
  link.href = URL.createObjectURL(blob);
  link.download = props.fileName;

  document.body.appendChild(link);
  link.click();
  document.body.removeChild(link);
  downloading.value = false;
}

async function handleRemove(del: boolean) {
  showLoader.value = true;
  if (!del) {
    showWarningPopup.value = false;
    showLoader.value = false;
    return;
  }
  if (!props.resourcePath) {
    showLoader.value = false;
    return;
  }
  let success;
  if (props.fileType === "folder") {
    success = await deleteFolder(props.resourcePath);
  } else {
    success = await deleteFile(props.resourcePath);
  }
  if (!success) {
    alert(`Couldn't delete ${props.fileType === "folder" ? "folder" : "file"}`);
    return;
  }
  refreshKey.value++;
  showWarningPopup.value = false;
  showLoader.value = false;
}

const show_rename = ref(false);
const new_name = ref(cutSuffix(props.fileName));

function handleRename() {
  if (
    (!userConfig.value?.permissions.RenameFiles &&
      props.fileType !== "folder") ||
    (!userConfig.value?.permissions.RenameFolders &&
      props.fileType === "folder")
  )
    return;
  show_rename.value = true;
}

async function handleAcceptRename() {
  showLoader.value = true;
  if (new_name.value === props.fileName) {
    show_rename.value = false;
    showLoader.value = false;
    return;
  }

  if (props.fileType !== "folder") {
    const res = await renameFile(props.resourcePath, new_name.value);
    showLoader.value = false;
    if (!res) {
      alert("Couldn't rename file");
    }
    refreshKey.value++;
    show_rename.value = false;
  } else {
    const res = await renameFolder(props.resourcePath, new_name.value);
    showLoader.value = false;
    if (!res) {
      alert("Couldn't rename folder");
      show_rename.value = false;
      return;
    }
    refreshKey.value++;
    show_rename.value = false;
  }
}

async function handleCancelRename() {
  show_rename.value = false;
}

const showWarningPopup = ref(false);
</script>

<template>
  <div
    class="file"
    :class="{ doubleView: doubleView }"
  >
    <div class="name-options">
      <div
        class="file-name"
        :class="props.fileType === 'folder' ? 'folder' : ''"
        @click="handleChangePath"
      >
        <FileIcons
          :name="fileName"
          :width="30"
          :height="30"
          :isFolder="fileType === `folder`"
          :isMulti="false"
          :isLink="false"
          :iconStyle="{ zIndex: 0 }"
        />
        <p
          :draggable="props.fileType !== 'folder' ? 'true' : 'false'"
          v-if="!show_rename"
          :title="fileName"
        >
          {{ props.fileName }}
        </p>
        <div v-if="show_rename" class="rename">
          <input type="text" v-model="new_name" autofocus />
          <img
            :src="accept_icon"
            alt="accept"
            id="accept"
            @click.stop="handleAcceptRename"
          />
          <img
            :src="cancel_icon"
            alt="cancel"
            id="cancel"
            @click.stop="handleCancelRename"
          />
        </div>
      </div>

      <div @dblclick.stop class="file-options-in-list">
        <img
          v-if="props.fileType !== 'folder'"
          :src="preview_icon"
          alt="preview"
          id="preview"
          @click="handlePreview(false)"
          :class="{ disabled: !userConfig?.permissions.Preview }"
          draggable="false"
        />
        <img
          v-if="props.fileType !== 'folder' && !downloading"
          :src="download_icon"
          id="download-img"
          alt="download"
          @click="
            () => {
              userConfig?.permissions.Download ? handleDownload() : null;
            }
          "
          draggable="false"
          :class="{ disabled: !userConfig?.permissions.Download }"
        />
        <div v-if="downloading" class="loader"></div>
        <img
          :src="rename_icon"
          alt="rename"
          @click="handleRename"
          draggable="false"
          id="rename-img"
          :class="{
            disabled:
              (!userConfig?.permissions.RenameFiles &&
                props.fileType !== 'folder') ||
              (!userConfig?.permissions.RenameFolders &&
                props.fileType === 'folder'),
          }"
        />
        <img
          draggable="false"
          :class="{
            disabled:
              (!userConfig?.permissions.DelFiles &&
                props.fileType !== 'folder') ||
              (!userConfig?.permissions.DelFolders &&
                props.fileType === 'folder'),
          }"
          :src="remove_icon"
          id="remove-img"
          alt="remove"
          @click="
            () => {
              if (
                (userConfig?.permissions.DelFiles &&
                  props.fileType !== 'folder') ||
                (userConfig?.permissions.DelFolders &&
                  props.fileType === 'folder')
              ) {
                showWarningPopup = true;
              }
            }
          "
        />
      </div>
    </div>
    <div class="size-info">
      <p v-if="props.fileType !== 'folder' && props.size">
        {{ formatBytes(parseInt(props.size)) }}
      </p>
    </div>
    <div class="created-info">
      <time :datetime="new Date(props.created).toISOString().split('T')[0]">
        {{ createdFormatted }}
      </time>
    </div>
    <div class="modified-info">
      <time :datetime="new Date(props.modified).toISOString().split('T')[0]">
        {{ modifiedFormatted }}
      </time>
    </div>

    <FilePreview
      v-if="previewWindow && resourcePath"
      :name="fileName"
      :resourcePath="resourcePath"
      :newTab="newTab"
      @closeCallback="
        () => {
          previewWindow = false;
        }
      "
    />
    <WarningPopup
      v-if="showWarningPopup"
      :title="`Are you sure you want to delete the ${
        fileType == 'folder' ? 'folder' : 'file'
      }`"
      :fileName="fileName"
      @closeCallback="
        (value) => {
          handleRemove(value);
        }
      "
    />
  </div>
</template>

<style lang="scss" scoped>
.non-draggable {
  pointer-events: none;
}

#preview-img{
  height: 1.5rem;
}

#download-img{
  height: 1.5rem;
}

#rename-img{
  height: 1.5rem;
}

#remove-img{
  height: 1.5rem;
}

.file {
  display: grid;
  grid-template-columns: 4fr 1fr 1fr 1fr;
  align-items: center;
  border-bottom: 1px solid #8f480a6b;
  cursor: default;

  .name-options {
    display: grid;
    grid-template-columns: minmax(0, 1fr) auto;
    align-items: center;
    padding: 0.6rem 1rem;

    .file-name {
      display: flex;
      gap: 1em;
      align-items: center;
      min-width: 0; // Prevents flex-basis from affecting layout
      max-width: 100%; // Ensures it only uses available space

      .rename {
        display: flex;
        gap: 0.5rem;

        img {
          width: 1.5rem;
          cursor: pointer;
        }

        input {
          border-radius: 0.3em;
          font-size: 1em;
          outline: 0;
          border: 1px solid gray;
          padding: 0.3em;
        }

        #accept:hover {
          filter: invert(21%) sepia(100%) saturate(7414%) hue-rotate(160deg)
            brightness(94%) contrast(117%);
        }

        #cancel:hover {
          filter: invert(21%) sepia(100%) saturate(7414%) hue-rotate(359deg)
            brightness(94%) contrast(117%);
        }
      }

      &.folder {
        cursor: pointer;
      }

      img {
        cursor: default;
        width: 2.5rem;
        flex-shrink: 0; // Prevents icon from shrinking
      }

      p {
        overflow: hidden;
        text-overflow: ellipsis;
        white-space: nowrap;
        flex-grow: 1;
        margin: 0;
        min-width: 0; // Ensures proper flex-shrink on text
        &:hover {
          cursor: pointer;
        }
      }
    }

    .file-options-in-list {
      display: flex;
      align-items: center;
      gap: 0.5rem;
      img {
        width: 1.5rem;
        cursor: pointer;
      }
    }
  }

  .size-info,
  .created-info,
  .modified-info {
    margin-left: 1rem;
    overflow: hidden;
    white-space: nowrap;
  }

  &:hover {
    background-color: #8f480a25;
  }

  .loader {
    width: 1.5rem;
    aspect-ratio: 1;
    border-radius: 50%;
    background: radial-gradient(farthest-side, #ffa516 94%, #0000) top/6px 6px
        no-repeat,
      conic-gradient(#0000 30%, #ffa516);
    -webkit-mask: radial-gradient(
      farthest-side,
      #0000 calc(100% - 6px),
      #000 0
    );
    animation: l13 1s infinite linear;
  }
  @keyframes l13 {
    100% {
      transform: rotate(1turn);
    }
  }

  &.doubleView {
    grid-template-columns: 1fr;

    .size-info,
    .created-info,
    .modified-info {
      display: none;
    }
  }
}
</style>
