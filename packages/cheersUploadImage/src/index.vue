<template>
	<div>
		<form method="post" enctype="multipart/form-data" style="display:none">
			<input :id="uuid" name="file" type="file" ref="userfile" @change="f_selectFile" accept="image/*" />
		</form>

		<div class="image-preview">
			<ul class="image-list">
				<template v-for="(file, index) in fileList">
					<li :style="imageThumbStyle" :key="index" v-if="fileList.length > 0">
						<image-cover :thumb="file"></image-cover>
						<close-btn class="delete-btn" @click.native="fileList.splice(index, 1)"></close-btn>
					</li>
				</template>
				<li class="flex-row flex-middle upload-image-button" v-if="fileList.length < multi">
					<label class="plus flex-item flex-row flex-center" :for="uuid">+</label>
				</li>
			</ul>
			<p class="img-font">{{text}}</p>
		</div>
		<div></div>

		<el-dialog :visible.sync="showCropper" title="选择图片区域" :close-on-click-modal="false" :close-on-press-escape="false"
			:show-close="false" :modal="false">
			<div class="image-cropper">
				<vue-cropper ref="cropper" guides :view-mode="2" drag-mode="move" autoCrop :min-container-width="150"
					:min-container-height="150" background rotatable scalable zoomable zoomOnTouch zoomOnWheel :src="imageSrc"
					:cropmove="f_cropImage"></vue-cropper>
			</div>
			<div slot="footer" class="dialog-footer">
				<el-button @click="f_cancelCrop">取 消</el-button>
				<el-button type="primary" @click="f_confirmCrop" :loading="uploading">确 定</el-button>
			</div>
		</el-dialog>
	</div>
</template>
<script>
import VueCropper from "vue-cropperjs";
import ImageCover from "./image-cover";
import CloseBtn from "./close-btn";
import FileAPI from "fileapi/dist/FileAPI.html5";
export default {
	name: "cheersUploadImage",
	components: {
		VueCropper,
		ImageCover,
		CloseBtn
	},
	props: {
		value: {
			type: String,
			default: ""
		},
		maxSize: {
			type: Number,
			default: 0
		},
		isCropper: {
			type: Boolean,
			default: true
		},
		// 文字备注
		text: {
			type: String,
			default: ""
		},
		localPreview: {
			type: Boolean,
			default: false
		},
		accept: {
			type: String,
			default: ""
		},
		title: {
			type: String,
			default: "上传文件"
		},
		width: {
			type: Number,
			default: 0
		},
		height: {
			type: Number,
			default: 0
		},
		multi: {
			type: Number,
			default: 1
		},
		uploadUrl: {
			type: "",
			default: () => ''
		}
	},
	data() {
		return {
			key: "",
			token: "",
			// url: window.api.qiniu,
			currentFile: {},
			fileList: [],
			imageSrc: "",
			cropImg: "",
			showCropper: false,
			uploadStatus: 0, // 0 未开始 1 上传中 2 已上传
			uploadPercent: 0,
			uploading: false
		};
	},
	computed: {
		imageThumbStyle() {
			return {
				width: `${this.width / 2}px`,
				height: `${this.height / 2}px`
			};
		},
		uuid() {
			return Math.random()
				.toString(36)
				.substring(3, 8);
		}
	},
	watch: {
		value: {
			immediate: true,
			handler: function () {
				let val = this.value;
				this.fileList = val && val.length ? val.split(",") : [];
			}
		},
		fileList: {
			deep: true,
			handler: function () {
				let val = this.fileList;
				this.$emit("input", val.length ? val.join(",") : "");
			}
		}
	},
	methods: {
		f_message(val) {
			this.$refs.userfile.value = null;
			this.$message({
				message: val,
				type: "warning"
			});
		},
		async f_selectFile(e) {
			let file = e.target.files[0];
			this.currentFile = file;
			let isAcceptType = new RegExp("image/*", "i").test(file.type);
			if (this.fileList.length >= this.multi) {
				return this.f_message(`最多上传${this.multi}个文件`);
			}
			if (!isAcceptType) {
				return this.f_message(`请选择图片类型文件`);
			}
			if (file.size > this.maxSize * 1000) {
				return this.f_message(`请上传小于${this.maxSize}KB的文件`);
			}
			if (this.isCropper) {
				this.f_previewFile(file);
				this.showCropper = true;
			} else {
				this.f_uploadFile(file);
			}
		},
		f_previewFile(file) {
			FileAPI.Image(file)
				.resize(800, 800, "max")
				.get((err, img) => {
					if (!err) {
						this.imageSrc = img.toDataURL();
						this.$refs.cropper.replace(this.imageSrc);
					}
				});
		},
		f_cropImage() {
			this.cropImg = this.$refs.cropper.getCroppedCanvas().toDataURL();
		},
		f_cancelCrop() {
			this.showCropper = false;
		},
		async f_confirmCrop() {
			this.uploading = true;
			let file = this.f_canvasToFile(
				this.$refs.cropper.getCroppedCanvas(),
				this.currentFile.name
			);
			if (this.localPreview) {
				var localUrl = window.URL.createObjectURL(file);
				this.fileList.push(localUrl);
				this.$emit("overUpload", false, file);
			} else {
				let imgUrl = await this.f_uploadFile(file);
			}
			this.uploading = false;
			this.showCropper = false;
		},
		f_canvasToFile(canvas, filename) {
			return new File(
				[this.f_dataURLtoBlob(canvas.toDataURL())],
				filename
			);
		},
		f_dataURLtoBlob(dataurl) {
			var arr = dataurl.split(",");
			var mime = arr[0].match(/:(.*?);/)[1];
			var bstr = atob(arr[1]);
			var n = bstr.length;
			var u8arr = new Uint8Array(n);
			while (n--) {
				u8arr[n] = bstr.charCodeAt(n);
			}
			return new Blob([u8arr], { type: mime });
		},

		// 上传
		async f_uploadFile(file, token, key) {
			var fd = new FormData();
			fd.append("file", file);
			let res = await this.$http.post(this.uploadUrl, fd, {
				headers: { "Content-Type": "multipart/form-data" }
			});
			if (res.body.errorCode == 0) {
				this.uploadStatus = 2;
				this.fileList.push(res.body.data.fileName);
			}
		}
	}
};
</script>
<style>
.upload-button > * {
	pointer-events: none;
}

.upload-button {
	display: block;
	width: 300px;
	min-height: 150px;
	margin: 10px auto auto auto;
	padding: 10px auto;
	border-radius: 10px;
	border: 2px dashed #ddd;
	overflow: hidden;
	cursor: pointer;
}

.upload-button span {
	display: block;
	width: 50px;
	height: 50px;
	border-radius: 100%;
	border: 3px solid #0983c7;
	margin: 30px auto 15px;
	position: relative;
}

.upload-button em {
	font-style: normal;
	text-align: center;
	display: block;
}

.upload-button span:before,
.upload-button span:after {
	display: block;
	content: "";
	background-color: #0983c7;
	position: absolute;
}

.upload-button span:before {
	width: 3px;
	height: 30px;
	top: 10px;
	left: 24px;
}

.upload-button span:after {
	width: 30px;
	height: 3px;
	top: 24px;
	left: 10px;
}

.audio-preview {
	width: 100%;
	position: relative;
}

.video-preview {
	width: 100%;
	position: relative;
}

.image-cropper {
	position: relative;
}

.cropper-buttons {
	position: fixed;
	bottom: 0;
	left: 0;
	right: 0;
	background-color: rgba(255, 255, 255, 0.6);
}

.btn {
	padding: 10px;
}

.btn-primary {
	background-color: rgba(235, 97, 0, 0.8);
	color: #fff;
}

/* 多图上传相关 */
.image-preview {
	position: relative;
}

.image-list {
	list-style: none;
	display: inline-block;
	margin: 0;
	padding: 0;
}

.image-list li {
	float: left;
	border: 1px solid #e8f5ff;
	position: relative;
	text-align: center;
	margin-right: 5px;
}

.upload-image-button {
	width: 75px;
	height: 75px;
	position: relative;
	border: 2px dashed #ddd !important;
	box-sizing: border-box;
}

.plus {
	font-size: 50px;
	color: #0983c7;
	line-height: 64px;
	margin-bottom: 15px;
	font-weight: 300;
	text-align: center;
	cursor: pointer;
	/* margin-left: 15px; */
}

.delete-btn {
	top: -8px;
	right: -5px;
	position: absolute !important;
	cursor: pointer;
}
.img-font {
	font-size: 12px;
	color: #666;
}
</style>
