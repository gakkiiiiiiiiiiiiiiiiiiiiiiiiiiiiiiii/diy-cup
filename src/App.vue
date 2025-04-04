<template>
	<div class="app-container">
		<div class="upload-container">
			<div>
				<input type="file" accept="image/*" @change="handleImageUpload" class="file-input" id="image-upload" />
				<label for="image-upload" class="upload-button"> 上传图片 </label>
				<canvas id="canvas" style="border: 1px solid #ccc"></canvas>
			</div>
		</div>
		<div class="canvas-container" ref="canvasContainer"></div>
	</div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount, watch } from 'vue';
import * as THREE from 'three';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls';
import { useWindowSize } from '@vueuse/core';
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader';
import * as fabric from 'fabric';

// 模型
const modelUrl = '/static/cup_of_coffee_gltf/scene.gltf';
// 贴图
const uvUrl = '/static/uv-2.png';

//绘制区域大小
const drawSize = {
	width: 1000,
	height: 500,
};

//绘制区域位于uv图的位置
const drawPosition = {
	x: 0,
	y: 0,
};

//绘制区域位于uv图的角度
const uvAngle = 180;

const canvasContainer = ref(null);
let scene, camera, renderer, controls, cup;
let fabricCanvas;

const uvBg = new Image();
uvBg.src = uvUrl;
uvBg.onload = (img) => {
	console.log('uvBg loaded');
};

const setupfabricCanvas = () => {
	fabricCanvas = new fabric.Canvas('canvas', {
		width: drawSize.width,
		height: drawSize.height,
	});
	fabricCanvas.renderAll();

	fabricCanvas.on('object:modified', (e) => {
		const url = fabricCanvas.toDataURL();
		uploadTextureByUrl(url);
	});
};

onMounted(() => {
	setupfabricCanvas();
});

const init = () => {
	// 创建场景
	scene = new THREE.Scene();
	scene.background = new THREE.Color(0xf5f5f5);

	// 创建相机
	camera = new THREE.PerspectiveCamera(75, (window.innerWidth * 0.5) / window.innerHeight, 0.1, 1000);
	camera.position.z = 5;

	// 创建渲染器
	renderer = new THREE.WebGLRenderer({ antialias: true });
	renderer.setSize(window.innerWidth * 0.5, window.innerHeight);
	renderer.setPixelRatio(window.devicePixelRatio);
	canvasContainer.value.appendChild(renderer.domElement);

	// 添加轨道控制器
	controls = new OrbitControls(camera, renderer.domElement);
	controls.enableDamping = true;
	controls.dampingFactor = 0.05;

	// 添加环境光和平行光
	const ambientLight = new THREE.AmbientLight(0xffffff, 0.5);
	scene.add(ambientLight);

	const directionalLight = new THREE.DirectionalLight(0xffffff, 0.8);
	directionalLight.position.set(5, 5, 5);
	scene.add(directionalLight);

	// 开始动画循环
	animate();
};

const loadImage = (url) => {
	return new Promise((resolve, reject) => {
		const image = new Image();
		image.src = url;
		image.onload = () => {
			console.log({ image });
			resolve(image);
		};
		image.onerror = (error) => {
			reject(error);
		};
	});
};

function drawRotatedImage(ctx, img, x, y, angle) {
	const x_ = (x - img.width) / 2;
	const y_ = (y - img.height) / 2;
	ctx.save(); // 保存画布状态
	ctx.translate(-x_, -y_); // 移动到 (x, y)
	ctx.rotate((angle * Math.PI) / 180); // 旋转角度（转换为弧度）
	ctx.drawImage(img, x_, y_); // 让图片中心对齐 (x, y)
	ctx.restore(); // 恢复画布状态
}

const jointUV = (url) => {
	return new Promise((resolve, reject) => {
		loadImage(url).then((insetImage) => {
			const bufferCanvas = document.createElement('canvas');
			const bufferCtx = bufferCanvas.getContext('2d');

			bufferCanvas.width = uvBg.width;
			bufferCanvas.height = uvBg.height;
			bufferCtx.drawImage(uvBg, 0, 0);

			drawRotatedImage(bufferCtx, insetImage, drawPosition.x, drawPosition.y, uvAngle);
			// bufferCtx.drawImage(insetImage, drawPosition.x, drawPosition.y);
			bufferCtx.restore();
			console.log({ bufferCtx });
			const uvUrl = bufferCanvas.toDataURL('image/png');
			resolve(uvUrl);
		});
	});
};

const animate = () => {
	requestAnimationFrame(animate);
	controls.update();
	renderer.render(scene, camera);
};

const uploadTextureByUrl = (url) => {
	jointUV(url).then((uvUrl) => {
		const texture = new THREE.TextureLoader().load(uvUrl);
		texture.wrapS = THREE.RepeatWrapping;
		texture.wrapT = THREE.RepeatWrapping;
		texture.image = uvUrl;
		cup.material.map = texture;
		cup.material.needsUpdate = true;
	});
};

const handleImageUpload = (event) => {
	const file = event.target.files[0];
	if (!file) return;

	const reader = new FileReader();
	reader.onload = (e) => {
		// 在fabricCanvas中添加上传的图片
		const blobUrl = URL.createObjectURL(file);

		fabric.Image.fromURL(blobUrl).then((img) => {
			fabricCanvas.add(img);
			fabricCanvas.renderAll();
			fabricCanvas.setActiveObject(img);

			const url = fabricCanvas.toDataURL();
			uploadTextureByUrl(url);
		});
	};
	reader.readAsDataURL(file);
};

const loadGLTFModel = () => {
	const loader = new GLTFLoader();

	loader.load(
		modelUrl,
		(gltf) => {
			cup = gltf.scene.children[0].children[0].children[0].children[0].children[0];

			cup.scale.set(0.01, 0.01, 0.01); // 缩小模型
			cup.position.set(0, 0, 0); // 将模型居中
			cup.rotation.set(Math.PI * 1.5, 0, 0); // 设置x轴旋转180度
			scene.add(cup);
			console.log({ cup });
			// 加载并应用zrn.jpg纹理
			const textureLoader = new THREE.TextureLoader();
			const texture = textureLoader.load(uvUrl);

			texture.wrapS = THREE.RepeatWrapping;
			texture.wrapT = THREE.RepeatWrapping;
			cup.material.map = texture;
			cup.material.needsUpdate = true;
		},
		undefined,
		(error) => {
			console.error('An error happened while loading the glTF model:', error);
		}
	);
};

const { width, height } = useWindowSize();

// 监听窗口大小变化
watch([width, height], () => {
	camera.aspect = (width.value * 0.5) / height.value;
	camera.updateProjectionMatrix();
	renderer.setSize(width.value * 0.5, height.value);
});

onMounted(() => {
	init();
	loadGLTFModel();
});

onBeforeUnmount(() => {
	// 清理资源
	scene.remove(cup);
	cup.geometry.dispose();
	cup.material.dispose();
	renderer.dispose();
});
</script>

<style scoped>
.app-container {
	display: flex;
	flex-direction: column;
	width: 100%;
	height: 100vh;
}

.upload-container {
	width: 100%;
	display: flex;
	justify-content: center;
	align-items: center;
	background-color: #f0f0f0;
}

.canvas-container {
	width: 100%;
	height: 100%;
}

.upload-section {
	background: white;
	padding: 1rem;
	border-radius: 8px;
	box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
}

.file-input {
	display: none;
}

.upload-button {
	display: inline-block;
	padding: 0.5rem 1rem;
	background-color: #4caf50;
	color: white;
	border-radius: 4px;
	cursor: pointer;
	transition: background-color 0.3s;
}

.upload-button:hover {
	background-color: #45a049;
}
</style>
