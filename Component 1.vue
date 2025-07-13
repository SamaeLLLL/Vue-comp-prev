<script setup lang="ts">
    import FileIcons from 'importPath'
    import { createApp, nextTick, ref, type VNodeRef } from 'importPath';
    import { VueFilesPreview } from 'importPath';
    import { downloadFile } from 'importPath';
    import { close_icon, new_tab_icon } from 'importPath';
    import { showLoader } from 'importPath';
    import { onMounted } from 'importPath';

    const filePreview = ref<InstanceType<typeof VueFilesPreview> | null>(null);
        const isPreviewEmpty = ref(false)
        onMounted(() => {
            checkPreviewable()
        })
        
        async function checkPreviewable() {
            await nextTick()  // Wait for DOM updates
            
            if (filePreview.value && filePreview.value.$el) {
                const previewElement = filePreview.value.$el
                return previewElement.children.length === 0
            }
            return true
        }
        
    const props = defineProps<{
        resourceParh: string,
        name: string,
        newTab: boolean
    }>()

    const emit = defineEmits(['closeCallback'])

    const uploadFile = ref();

    async function downloadPreview() {
        showLoader.value = true;
        const fileBlob = await downloadFile(props.resourceParh);
        if(fileBlob) {
            uploadFile.value = new File([fileBlob], props.name, {type: fileBlob.type})
        }
        const previewable = await checkPreviewable()
        isPreviewEmpty.value = previewable;
        showLoader.value = false;
        if(!props.newTab && getFileType(uploadFile.value) !== "EML") return;
        popupPreview()
        emit('closeCallback')
    }
    // Call the download function
    downloadPreview()

    function getFileType(file:File) {
    if (!file) return "Unknown";

    const mimeType = file.type; // e.g., "image/png", "application/pdf"
    const extension = file?.name?.split('.').pop()?.toLowerCase() || ''; // Get file extension

    // Define categories based on MIME types
    if (mimeType.startsWith("image/")) return "Image";
    if (mimeType === "application/pdf") return "PDF";
    if (mimeType === "application/vnd.openxmlformats-officedocument.wordprocessingml.document") return "DOCX";
    if (mimeType === "text/plain") return "TXT";

    // Handle cases where the MIME type might be missing
    const knownExtensions: { [key: string]: string } = {
        "jpg": "Image",
        "jpeg": "Image",
        "png": "Image",
        "gif": "Image",
        "bmp": "Image",
        "webp": "Image",
        "pdf": "PDF",
        "docx": "DOCX",
        "txt": "TXT",
        "eml": "EML"
    };

    return knownExtensions[extension] || "Other";
}

function fileToBase64(file: File): Promise<string> {
  return new Promise((resolve, reject) => {
    const reader = new FileReader();

    reader.onloadend = () => {
      // Extract base64 string (without the data URL prefix)
      const base64String = (reader.result as string).split(',')[1];
      resolve(base64String);
    };
    
    reader.onerror = (error) => reject(error);

    reader.readAsDataURL(file);
  });
}

function base64ToArrayBuffer(base64:string): ArrayBuffer {
  const binaryString = atob(base64); // Decode base64 to binary string
  const len = binaryString.length;
  const bytes = new Uint8Array(len);
  for (let i = 0; i < len; i++) {
    bytes[i] = binaryString.charCodeAt(i);
  }
  return bytes.buffer; // This is the ArrayBuffer
}


async function popupPreview() {
    const previewWindow = window.open('', '_blank', 'resizable=yes,scrollbars=yes');
    if (!previewWindow) return;

    const fileUrl = URL.createObjectURL(uploadFile.value);
    const fileType = getFileType(uploadFile.value); // Should return "Image", "PDF", "Text", or "DOCX"
    let message = "File type cannot be previewed";
    let attUrls = [];

    const fileBase = await fileToBase64(uploadFile.value)
    let fileHTML;
    let safeHTML;
    let details = {
        date: "",
        from: "",
        to: "",
        subject: "",
    }
    if(fileType === "EML") {
        // @ts-ignore
        const emlr = await import('../util/EmlReader');
        const buffer = base64ToArrayBuffer(fileBase)
        const email = new emlr.EmlReader(buffer);
        details.date = email.getDate();
        details.from = email.getFrom();
        details.to = email.getTo();
        details.subject = email.getSubject();

        const attachments = email.getAttachments();

        attUrls = attachments
            .map((attachment: any) =>
                URL.createObjectURL(new Blob([attachment.content], { type: attachment.contentType }))
            ).join('|');
        message = "This EML file contains unparsable content and is unpreviewable. You can download it and open it in your email client.";
        fileHTML = email.getMessageHtml();
        safeHTML = JSON.stringify(fileHTML);
    }
    
    // Write the basic HTML structure into the preview window
    // As far as I know / am concerned, this is the only valid way to create a popup in an embedded environment. It's quite limiting but works well enough
    previewWindow.document.write(`
        <html>
            <head>
                <title>File Preview</title>
                <style>
                    body {
                        font-family: sans-serif;
                        display: flex;
                        justify-content: center;
                        height: 100vh;
                        margin: 0;
                        box-sizing: border-box;
                    }
                    .preview {
                        max-width: 100%;
                        max-height: 100%;
                        overflow: auto;
                    }
                    iframe.preview {
                        width: 100vw;
                        height: 100vh;
                        border: none;
                    }
                    pre.preview {
                        white-space: pre-wrap;
                        word-wrap: break-word;
                    }
                </style>
            </head>
            <body>
                <p>${message}</p>
                <script>
                    window.onload = async function() {

                        async function base64ToFile(base64, filename, mimeType) {
                            // Decode the base64 string
                            const byteCharacters = atob(base64);
                            
                            // Convert the decoded byte string into an array of bytes
                            const byteArrays = new Uint8Array(byteCharacters.length);
                            
                            for (let i = 0; i < byteCharacters.length; i++) {
                                byteArrays[i] = byteCharacters.charCodeAt(i);
                            }
                            
                            // Create and return a File object
                            return new File([byteArrays], filename, { type: mimeType });
                        }

                        const fileUrl = "${fileUrl}";
                        const file = "${fileBase}";
                        const fileName = "${props.name}";
                        const fileType = "${fileType}";
                        const aUrls = "${attUrls}".split('|');

                        const app = document.body;
                        app.innerHTML = ""; // Clear the loading message

                        if (fileType === "Image") {
                            const img = document.createElement("img");
                            img.src = fileUrl;
                            img.className = "preview";
                            app.appendChild(img);
                        } else if (fileType === "PDF") {
                            const iframe = document.createElement("iframe");
                            const filePDF = await base64ToFile(file, fileName, "application/pdf");
                            const fileUrlPDF = URL.createObjectURL(filePDF);
                            iframe.src = fileUrlPDF;
                            iframe.className = "preview";
                            app.appendChild(iframe);
                        } else if (fileType === "TXT") {
                            // Load and display a text file
                            fetch(fileUrl)
                                .then(response => response.text())
                                .then(text => {
                                    const pre = document.createElement("pre");
                                    pre.className = "preview";
                                    pre.textContent = text;
                                    app.appendChild(pre);
                                })
                                .catch(err => {
                                    app.innerHTML = "<p>Failed to load text file.</p>";
                                });
                        } else if (fileType === "DOCX") {
                            // Load Mammoth.js to convert DOCX to HTML
                            const script = document.createElement("script");
                            script.src = "https://cdnjs.cloudflare.com/ajax/libs/mammoth/1.4.2/mammoth.browser.min.js";
                            script.onload = function() {
                                fetch(fileUrl)
                                    .then(response => response.arrayBuffer())
                                    .then(arrayBuffer => {
                                        mammoth.convertToHtml({ arrayBuffer: arrayBuffer })
                                            .then(function(result) {
                                                const div = document.createElement("div");
                                                div.innerHTML = result.value;
                                                div.className = "preview";
                                                div.style.backgroundColor = "white";
                                                div.style.padding = "1em";
                                                div.style.width = "210mm";
                                                app.appendChild(div);
                                            })
                                            .catch(function(error) {
                                                app.innerHTML = "<p>Failed to preview DOCX file.</p>";
                                            });
                                    })
                                    .catch(err => {
                                        app.innerHTML = "<p>Failed to load DOCX file.</p>";
                                    });
                            }
                            document.head.appendChild(script);
                            document.body.style.backgroundColor = "gray";
                        } else if (fileType === "EML") {
                            const emlDiv = document.createElement("div");
                            emlDiv.style.padding = "1em";
                            emlDiv.innerHTML = JSON.parse(${JSON.stringify(safeHTML)});

                            const div = document.createElement("div");

                            const pFrom = document.createElement("p");
                            const bFrom = document.createElement("b");
                            bFrom.textContent = "From: ";
                            pFrom.appendChild(bFrom);
                            pFrom.append("${details.from}");
                            div.appendChild(pFrom);

                            const pTo = document.createElement("p");
                            const bTo = document.createElement("b");
                            bTo.textContent = "To: ";
                            pTo.appendChild(bTo);
                            pTo.append("${details.to}");
                            div.appendChild(pTo);
                            div.innerHTML += "<p><b>Date:</b> ${details.date}</p>";
                            div.innerHTML += "<p><b>Subject: ${details.subject}</b><p>";
                            div.style.padding = "1em";
                            div.style.backgroundColor = "#fffaf4";
                            div.querySelectorAll("*").forEach(child => {
                                child.style.setProperty("color", "black", "important");
                            });
                            div.style.borderBottom = "1px solid #c1b6ad";
                            document.body.appendChild(div);
                            document.body.appendChild(emlDiv);
                            document.body.style.flexDirection = "column";
                            document.body.style.justifyContent = "flex-start";
                            
                            document.body.style.height = "auto";

                            for(let i = 0; i < aUrls.length; i++) {
                                const attUrl = aUrls[i];
                                if(attUrl) {
                                    console.log("test");
                                    const img = document.createElement("img");
                                    img.src = attUrl;

                                    app.appendChild(img);
                                }
                            }
                        } else {
                            app.innerHTML = "<p>File type cannot be previewed.</p>";
                        }
                    };
                </scr` + `ipt>
            </body>
        </html>
    `);
    // Need to split the script tag or won't compile (line 315)

    previewWindow.document.close();

    // Free memory when the preview window is closed
    previewWindow.onbeforeunload = () => URL.revokeObjectURL(fileUrl);
}


</script>

<template>
    <div class="popup">
        <div class="preview">
            <div class="top-bar">
                <div class="left">
                    <FileIcons 
                        :name="name" :width="30" :height="30"
                        :isLink="false"
                     />
                    <p>{{ name }}</p>
                </div>
                <img class="close" :src="close_icon" alt="" @click="() => {emit('closeCallback')}">
            </div>
            <div v-if="uploadFile" class="preview-container">
                <VueFilesPreview v-if="!isPreviewEmpty" ref="filePreview" :height="`80vh`" class="file-preview" :file="uploadFile" />
                <p class="msg" v-else>File type cannot be previewed</p>
            </div>

        </div>
    </div>
</template>


<style lang="scss">
    .popup {
        position: fixed;
        font-size: 1em;
        left: 0;
        top: 0;
        width: 100vw;
        height: 100vh;
        background-color: rgba(0, 0, 0, 0.300);
        z-index: 20;
    }

    .preview {
        min-width: 50vw;
        max-width: 80vw;
        background-color: white;
        position: fixed;
        top: 50%;
        left: 50%;
        padding: 2em;
        border-radius: .5em;
        transform: translate(-50%, -50%);
        min-height: 50vh;
        .top-bar {
            .left {
                display: flex;
                align-items: center;
                gap: 1em;

                .divider-vertical {
                    width: 1px;
                    height: 1.5em;
                    background-color: rgba(0, 0, 0, 0.445);
                }

                .tab-img {
                    width: 1.5em;
                    &:hover {
                        cursor: pointer;
                        scale: 1.05;
                    }
                }
            }
            display: flex;
            align-items: center;
            justify-content: space-between;
            margin: 0 1em;
            margin-bottom: 2em;
            img {
                width: 2em;
            }
        }
        .close {
            cursor: pointer;
        }
        .file-preview {
            overflow-y: scroll;
            // >div {
            //     // display: flex;
            //     // justify-content: center;
            // }
        }
        .preview-container {
            .msg {
                text-align: center;
                margin-top: 4em;
                color: rgb(199, 199, 199);
            }
        }
        img {
            max-width: 80vw;
            max-height: 79vh;
            object-fit: cover;
        }
    }

</style>
