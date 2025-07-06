<script setup>
import {reactive, ref, watch} from 'vue';
import {message} from 'ant-design-vue';
import {get, post} from '@/utils/request.js'
// 响应式状态
const taskContent = ref('');
const screenshot = ref(null);
const rawModelResponse = ref(null);
const modifiedResult = ref('');// 不再需要
const initLoading = ref(false);
const modelLoading = ref(false);
const taskTypes = [{ label: '购物', value: 'shopping' },{label:'消息',value:'news'}];
const applications = {
  shopping: [{ label: '美团', value: 'meituan_waimai' }],
  // 你可以继续添加其他类型和应用
  news:[{label:'微信',value:'wechat'}]
};
const selectedTaskType = ref('shopping');
const selectedApplication = ref('meituan_waimai');
const shoppingTaskParameters = reactive({
  count: 0,
  product_name: '',
  store_name: '',
  specifications: ''
});
// --- [新增] 表单相关的状态和常量 ---
// Action 的所有可能类型
const actionTypes = [
  { label: '点击 (tap)', value: 'tap' },
  { label: '输入文本 (text)', value: 'text' },
  { label: '需要反馈 (need_feedback)', value: 'need_feedback' },
  { label: '长按 (long_press)', value: 'long_press' },
  { label: '滑动 (swipe)', value: 'swipe' },
  { label: '滑动 (swipe_two_points)', value: 'swipe_two_points' },
  { label: '等待 (wait)', value: 'wait' },
  { label: '结束 (FINISH)', value: 'FINISH' },
];

// Swipe 方向和距离的选项
const swipeDirections = [
  { label: '上', value: 'up' },
  { label: '下', value: 'down' },
  { label: '左', value: 'left' },
  { label: '右', value: 'right' },
];
const swipeDistances = [
  { label: '短', value: 'short' },
  { label: '中', value: 'medium' },
  { label: '长', value: 'long' },
];

// 用于驱动表单的响应式对象
const actionForm = reactive({
  action: null,
  x: null,
  y: null,
  value: '',
  direction: 'up',
  distance: 'medium',
  x_end: null,
  y_end: null
});

/**
 * 将给定的图片URL转换为Base64编码的Data URL。
 *
 * @param {string} imageUrl 要转换的图片URL。
 * @returns {Promise<string>} 一个Promise，解析为Base64字符串（data:image/jpeg;base64,...），如果失败则reject。
 */
const convertImageToBase64 = (imageUrl) => {
  return new Promise(async (resolve, reject) => {
    try {
      // 1. 使用 fetch API 获取图片数据
      const response = await fetch(imageUrl);

      if (!response.ok) {
        // 如果HTTP请求不成功，抛出错误
        alert(`HTTP error! Status: ${response.status} for URL: ${imageUrl}`);
      }

      // 2. 获取 Blob 对象
      const imageBlob = await response.blob();

      // 3. 使用 FileReader 将 Blob 转换为 Base64
      const reader = new FileReader();

      reader.onloadend = () => {
        // FileReader 读取完成后，result 就是 Base64 格式的 Data URL
        resolve(reader.result);
      };

      reader.onerror = () => {
        // FileReader 发生错误时，reject Promise
        reject(new Error(`FileReader failed to read the image blob from ${imageUrl}.`));
      };

      reader.readAsDataURL(imageBlob); // 读取 Blob 为 Data URL (Base64)

    } catch (err) {
      // 捕获 fetch 或其他异步操作中的错误，并 reject Promise
      reject(new Error(`Failed to fetch or process image "${imageUrl}": ${err.message}`));
    }
  });
};
// 初始化app
const initApp= async ()=>{
  await post('/restart-app');

  // 定义要等待的时间（毫秒）
  const delay = 5000; // 5秒
// 使用 setTimeout 进行延时执行
  setTimeout(() => {
    // 在这里放置你希望延迟执行的代码
    getScreenshot()
  }, delay);

  // 加载截图-adb未成功前的临时处理
  // console.log(screenshot_base64);
  // screenshot.value = await convertImageToBase64('src/assets/screenshots/01_main_page.png');
}
const testImageLoad=async ()=>{
  screenshot.value = await convertImageToBase64('src/assets/screenshots/test1.jpeg');
}
/**
 * [新增] 一个辅助函数，用于根据后端返回的 action 对象更新表单
 * @param {object} actionObject - 从后端获取的 action JSON 对象
 */
const updateActionForm = (actionObject) => {
  if (!actionObject || !actionObject.action) {
    // 如果没有数据，重置表单
    actionForm.action = null;
    actionForm.x = null;
    actionForm.y = null;
    actionForm.value = '';
    actionForm.direction = 'up';
    actionForm.distance = 'medium';
    actionForm.x_end = null;
    actionForm.y_end = null;
    return;
  }
  // 使用 Object.assign 来填充表单，后端返回的对象可能不包含所有字段
  // 后面属性的优先级更高
  Object.assign(actionForm, {
    // 先设置默认值
    action: null,
    x: null,
    y: null,
    value: '',
    direction: 'up',
    distance: 'medium',
    x_end: null,
    y_end: null,
    // 再用后端数据覆盖
    ...actionObject
  });
};
// --- [新增] 用于存储鼠标在图片上坐标的状态 ---
const mouseCoordinates = ref({ x: 0, y: 0 });
// [新增] 用于存储图片原始分辨率的状态
const originalImageSize = ref({ width: 0, height: 0 });
const showImageSize = ref({ width: 0, height: 0 });// 显示的图片尺寸
// [新增] 用于获取截图包裹容器DOM元素的模板引用
const screenshotWrapperRef = ref(null);
// --- [新增] Canvas 的模板引用 ---
const canvasRef = ref(null);
const user_id = ref('');// 用户id,用于将标注保存到不同的标注中
/**
 * [重构] 核心绘图函数：重绘基础截图
 * 负责清空Canvas并绘制当前的截图。这是所有预览和重置操作的基础。
 */
const redrawBaseImage = () => {
  const canvas = canvasRef.value;
  const src = screenshot.value;

  if (!canvas) return;
  const ctx = canvas.getContext('2d');

  // 如果没有截图，则清空画布
  if (!src) {
    ctx.clearRect(0, 0, canvas.width, canvas.height);
    originalImageSize.value = { width: 0, height: 0 };
    return;
  }

  const img = new Image();
  img.onload = () => {
    // 首次加载或图片变化时，设置Canvas尺寸和缩放
    if (originalImageSize.value.width !== img.width || originalImageSize.value.height !== img.height) {
      originalImageSize.value = { width: img.width, height: img.height };

      const dpr = window.devicePixelRatio || 1;
      const displayHeight = canvas.clientHeight; // 从CSS获取显示高度
      const displayWidth = displayHeight * img.width / img.height;

      // 设置Canvas的绘图缓冲尺寸，以保证高清
      canvas.width = displayWidth * dpr;
      canvas.height = displayHeight * dpr;

      // 缩放Canvas上下文，使得后续绘图可以使用CSS像素单位
      ctx.scale(dpr, dpr);
      ctx.imageSmoothingEnabled = true;
      ctx.imageSmoothingQuality = 'high';
    }

    // 执行绘图
    const displayWidth = canvas.clientWidth;
    const displayHeight = canvas.clientHeight;
    ctx.clearRect(0, 0, displayWidth, displayHeight);
    ctx.drawImage(img, 0, 0, displayWidth, displayHeight);
  };
  img.onerror = () => console.error("无法加载图片用于绘制。");
  img.src = src;
};

/**
 * [新增] 使用 watch 监听 screenshot 的变化。
 * 当截图加载后，自动计算其原始分辨率。
 */
watch(screenshot, (newScreenshot) => {
  redrawBaseImage();
});



// 推理
const infer=async ()=>{
  const res= await post('/infer',{
    'image_base64': screenshot.value,
    'slot_info': {
      'task_type':selectedTaskType.value,
      'app_name':selectedApplication.value,
      'quantity':shoppingTaskParameters.count,
      'item_name':shoppingTaskParameters.product_name,
      'store_name':shoppingTaskParameters.store_name,
      'specs':shoppingTaskParameters.specifications
    }
  });
  if(res.status==='success'){
    console.log('模型响应',res.data);
    rawModelResponse.value = res.data; // 保存原始响应
    updateActionForm(res.data); // 使用新函数更新我们的表单
    // 预览标注
    handlePreview();
  }
}
// 获取截图
const getScreenshot=async ()=>{
  try {
    const res= await get('/get-screenshot');
    if(res.status === 'success'){
      const screenshot_base64=res.data;
      console.log(screenshot_base64);
      screenshot.value=screenshot_base64;
    }
  }catch(err){
    alert(err);
  }
}
// 生成任务
const generateTask = () => {
  shoppingTaskParameters.count=2;
  shoppingTaskParameters.product_name='多肉葡萄';
  shoppingTaskParameters.store_name='喜茶';
  shoppingTaskParameters.specifications='';
}
// 保存标注
const handleSaveAnnotation = async () => {
  try {
    await post('/save-annotation', {
      'image_base64': screenshot.value,
      'slot_info': {
        'task_type':selectedTaskType.value,
        'app_name':selectedApplication.value,
        'quantity':shoppingTaskParameters.count,
        'item_name':shoppingTaskParameters.product_name,
        'store_name':shoppingTaskParameters.store_name,
        'specs':shoppingTaskParameters.specifications
      },
      'action_info':actionForm,
      'user_id':user_id.value,

    });
    console.log(actionForm.value);
    // // 执行真实操作
    await post('/execute-action', {
      action_info: actionForm
    });
    message.success('标注保存成功');
    // 清空标注
    Object.assign(actionForm, {
      // 设置默认值
      action: null,
      x: null,
      y: null,
      value: '',
      direction: 'up',
      distance: 'medium',
      x_end: null,
      y_end: null,
    });
    // 获取应用截图
    await getScreenshot();
    // 清空过时的ActionForm

    // 获取应用截图
    // setTimeout(()=>{
    //   getScreenshot();
    // },1500)
  } catch (error) {
    message.error(`保存失败: ${error.message}`);
  }
    // message.success('标注保存成功');
    // screenshot.value=await convertImageToBase64('src/assets/screenshots/02_search_store.png');
};
/**
 * [新增] 封装后的坐标换算函数 (纯函数)
 * 将鼠标在元素上的显示坐标转换为相对于图片原始分辨率的真实坐标。
 *
 *  @param {number} mouseX 鼠标在 Canvas 元素上的 X 坐标（显示坐标）。
 *  @param {number} mouseY 鼠标在 Canvas 元素上的 Y 坐标（显示坐标）。
 * @returns {{x: number, y: number} | null} - 返回计算后的真实坐标对象，如果无法计算则返回 null。
 */
/**
 * [优化] 坐标转换：将显示坐标转换为真实坐标 (修复了计算bug)
 */
const convertDisplayCoordsToReal = (mouseX, mouseY) => {
  const canvas = canvasRef.value;
  if (!canvas || !originalImageSize.value.width) return null;

  const displayW = canvas.clientWidth;
  const displayH = canvas.clientHeight;
  const originalW = originalImageSize.value.width;
  const originalH = originalImageSize.value.height;

  if (displayW === 0 || displayH === 0) return null;

  const realX = Math.round((mouseX / displayW) * originalW);
  const realY = Math.round((mouseY / displayH) * originalH);

  return { x: realX, y: realY };
};

// --- [新增] 鼠标在截图上移动的事件处理函数 ---
/**
 * 处理鼠标在截图图片上移动的事件。
 * @param {MouseEvent} event - 鼠标事件对象
 */
const handleScreenshotMousemove = (event) => {
  // event.offsetX 和 event.offsetY 能直接获取鼠标相对于事件源元素（这里是图片）左上角的坐标
  // 调用封装好的函数进行计算
  // 更新状态
  mouseCoordinates.value = convertDisplayCoordsToReal(
      event.offsetX,
      event.offsetY,
  );
};

/**
 * [新增] 当鼠标移出图片区域时，重置坐标显示
 */
const handleScreenshotMouseleave = () => {
  mouseCoordinates.value = { x: 0, y: 0 }; // 或者重置为 { x: 0, y: 0 }，设为null可以隐藏显示
}
/**
 * [新增] 点击 Canvas 时，自动填充坐标到表单中。
 * 这是一个非常实用的交互增强。
 */
const handleCanvasClick = (event) => {
  if (actionForm.action === 'tap' || actionForm.action === 'long_press' || actionForm.action === 'swipe') {
    // 之前这里传递的参数不正确，已修复
    const coords = convertDisplayCoordsToReal(
        event.offsetX,
        event.offsetY
    );
    if (coords) {
      actionForm.x = coords.x;
      actionForm.y = coords.y;
      message.success(`坐标已更新: (${coords.x}, ${coords.y})`);
      // 更新标注
      handlePreview();
    }
  }if(actionForm.action === 'swipe_two_points' ) {
    const coords = convertDisplayCoordsToReal(
        event.offsetX,
        event.offsetY
    );
    // 如果初始坐标存在则填充初始坐标
    if(!actionForm.x||!actionForm.y){
      if(coords) {
        actionForm.x = coords.x;
        actionForm.y = coords.y;
        message.success(`起始坐标已更新: (${coords.x}, ${coords.y})`);
      }
    }else{
      // 填充末尾坐标
      // 相对距离长的轴决定方向,另一个轴决定距离
      const x_distance = Math.abs(coords.x - actionForm.x);
      const y_distance = Math.abs(coords.y - actionForm.y);
      if(x_distance >= y_distance){
        actionForm.x_end = coords.x;
        actionForm.y_end = actionForm.y;
      }else{
        actionForm.x_end = actionForm.x;
        actionForm.y_end = coords.y;
      }
      message.success(`末尾坐标已更新: (${actionForm.x_end}, ${actionForm.y_end})`);
      handlePreview();
    }
  }
};
/**
 * [新增] 预览操作处理函数
 * 这是“预览操作”按钮的核心逻辑。
 */
const handlePreview = () => {
  const action = actionForm.action;
  if (!screenshot.value || !action) {
    message.warn('请先加载截图并生成一个操作');
    return;
  }

  // 1. 先重绘基础图片，清除旧的预览
  redrawBaseImage();

  const canvas = canvasRef.value;
  // 等待图片绘制完成后再画标注
  setTimeout(() => {
    const ctx = canvas.getContext('2d');

    // 2. 根据操作类型绘制标注
    if (action === 'tap' || action === 'long_press') {
      if (actionForm.x == null || actionForm.y == null) {
        message.warn('预览失败：坐标不完整');
        return;
      }
      const displayCoords = convertRealCoordsToDisplay(actionForm.x, actionForm.y);
      if (displayCoords) {
        drawTapOnCanvas(ctx, displayCoords.x, displayCoords.y);
      }
    } else if (action === 'swipe') {
      if (actionForm.x == null || actionForm.y == null) {
        message.warn('预览失败：起始坐标不完整');
        return;
      }
      const startCoords = convertRealCoordsToDisplay(actionForm.x, actionForm.y);
      if (startCoords) {
        // 定义不同距离在屏幕上的像素长度
        const distanceMap = { short: 80, medium: 150, long: 220 };
        const length = distanceMap[actionForm.distance] || 150;

        let endX = startCoords.x;
        let endY = startCoords.y;

        switch (actionForm.direction) {
          case 'up':    endY -= length; break;
          case 'down':  endY += length; break;
          case 'left':  endX -= length; break;
          case 'right': endX += length; break;
        }
        drawTapOnCanvas(ctx, startCoords.x, startCoords.y);
        drawSwipeArrowOnCanvas(ctx, startCoords.x, startCoords.y, endX, endY);
      }
    } else if (action === 'swipe_two_points') {
      if (actionForm.x == null || actionForm.y == null) {
        message.warn('预览失败：起始坐标不完整');
        return;
      }
      if(actionForm.x_end == null || actionForm.y_end == null) {
        message.warn('预览失败: 终止坐标不完整');
        return;
      }
      const startCoords = convertRealCoordsToDisplay(actionForm.x, actionForm.y);
      const endCoords = convertRealCoordsToDisplay(actionForm.x_end, actionForm.y_end);
      if (startCoords&&endCoords) {
        drawTapOnCanvas(ctx, startCoords.x, startCoords.y);
        drawSwipeArrowOnCanvas(ctx, startCoords.x, startCoords.y, endCoords.x, endCoords.y);
      }
    } else {
      message.info('该操作类型不支持预览');
    }
  }, 100); // 短暂延迟确保图片已绘完
};

/**
 * [新增] 绘图工具：在Canvas上画十字圆环
 */
const drawTapOnCanvas = (ctx, x, y) => {
  ctx.beginPath();
  ctx.strokeStyle = '#FF4D4F'; // 使用醒目的红色
  ctx.lineWidth = 1.5;

  // 画圆
  const radius = 10;
  ctx.arc(x, y, radius, 0, 2 * Math.PI);

  // 画十字
  ctx.moveTo(x - radius - 8, y);
  ctx.lineTo(x + radius + 8, y);
  ctx.moveTo(x, y - radius - 8);
  ctx.lineTo(x, y + radius + 8);

  ctx.stroke();
};

/**
 * [新增] 绘图工具：在Canvas上画箭头
 */
const drawSwipeArrowOnCanvas = (ctx, fromX, fromY, toX, toY) => {
  ctx.beginPath();
  ctx.strokeStyle = '#110a01'; // 使用醒目的绿色
  ctx.fillStyle = '#0c0c0b';
  ctx.lineWidth = 2;

  // 画线
  ctx.moveTo(fromX, fromY);
  ctx.lineTo(toX, toY);

  // 画箭头
  const headlen = 15; // 箭头头的长度
  const angle = Math.atan2(toY - fromY, toX - fromX);

  ctx.moveTo(toX, toY);
  ctx.lineTo(toX - headlen * Math.cos(angle - Math.PI / 6), toY - headlen * Math.sin(angle - Math.PI / 6));
  ctx.moveTo(toX, toY);
  ctx.lineTo(toX - headlen * Math.cos(angle + Math.PI / 6), toY - headlen * Math.sin(angle + Math.PI / 6));

  ctx.stroke();
};


/**
 * [新增] 坐标转换：将真实坐标转换为Canvas显示坐标
 */
const convertRealCoordsToDisplay = (realX, realY) => {
  const canvas = canvasRef.value;
  if (!canvas || !originalImageSize.value.width) return null;

  const displayW = canvas.clientWidth;
  const displayH = canvas.clientHeight;
  const originalW = originalImageSize.value.width;
  const originalH = originalImageSize.value.height;

  const displayX = (realX / originalW) * displayW;
  const displayY = (realY / originalH) * displayH;

  return { x: displayX, y: displayY };
};
// 重置修改
const handleReset = () => {
  updateActionForm(rawModelResponse.value);
  redrawBaseImage(); // 清除画布上的所有预览标注
  message.info('已重置为原始推理结果');
};
// 测试截图
const testRestart= async ()=>{
  try {
    const res = await post('/restart-app');
    if(res.status === 'status'){
      message.success('成功重启')
    }
  }catch(error){
    console.error(error);
  }


}
</script>

<template>
  <a-layout class="container">
    <!-- 任务输入区 -->
<!--    <a-layout-header>-->
<!--      <a-space>-->
<!--        <a-input-->
<!--            v-model:value="taskContent"-->
<!--            placeholder="请输入任务内容"-->
<!--            style="width: 800px"-->
<!--        />-->
<!--        <a-button-->
<!--            type="primary"-->
<!--            :loading="initLoading"-->
<!--            @click="handleInitTask"-->
<!--        >-->
<!--          初始化任务-->
<!--        </a-button>-->

<!--      </a-space>-->
<!--    </a-layout-header>-->
    <!-- 在任务输入区部分加入（可以在按钮旁边） -->
    <a-space style="margin-bottom: 16px;">
      <!-- 第一级：任务类型 -->
      <a-select
          v-model:value="selectedTaskType"
          :options="taskTypes"
          style="width: 120px"
          placeholder="选择任务类型"
          @change="()=>{selectedApplication=''}"
      />
      <!-- 第二级：应用 -->
      <a-select
          v-model:value="selectedApplication"
          :options="applications[selectedTaskType]"
          style="width: 120px"
          placeholder="选择应用"
      />
      用户id:
      <a-input v-model:value="user_id" placeholder="用户id">
      </a-input>
      <a-button
          type="primary"
          :loading="initLoading"
          @click="generateTask"
      >
        生成任务
      </a-button>
      <a-button
          type="primary"
          :loading="initLoading"
          @click="initApp"
      >
        重启app,截图
      </a-button>
      <a-button
          type="primary"
          :loading="initLoading"
          @click="infer"
      >
        请求模型,生成操作
      </a-button>
      <a-button
          type="primary"
          :loading="initLoading"
          @click="handleSaveAnnotation"
          :disabled="!actionForm.action"
      >
        保存结果,执行,截图
      </a-button>
      <a-button
          type="primary"
          :loading="initLoading"
          @click="getScreenshot"
      >
        截图
      </a-button>
      <a-button
          type="primary"
          :loading="initLoading"
          @click="testImageLoad"
          v-if="false"
      >
        测试截图
      </a-button>
      <a-button
          type="primary"
          :loading="initLoading"
          @click="testRestart"
          v-if="false"
      >
        测试重启app
      </a-button>
    </a-space>

    <!-- 参数输入部分-shopping -->
    <a-form layout="inline" v-if="selectedTaskType==='shopping'">
      <a-form-item label="商品数量">
        <a-input-number v-model:value="shoppingTaskParameters['count']" style="width: 60px"/>
      </a-form-item>
      <a-form-item label="商品名称">
        <a-input v-model:value="shoppingTaskParameters['product_name']" placeholder="请输入商品名称" />
      </a-form-item>
      <a-form-item label="店铺名称">
        <a-input v-model:value="shoppingTaskParameters['store_name']" placeholder="请输入店铺名称" />
      </a-form-item>
      <a-form-item label="商品规格">
        <a-input v-model:value="shoppingTaskParameters['specifications']" placeholder="请输入商品规格" />
      </a-form-item>

    </a-form>

    <!-- 主要内容区 -->
    <a-layout-content class="content">
      <a-row :gutter="24">
        <!-- 截图展示列 -->
        <a-col :span="12">
          <a-card title="云手机截图" >
            <!-- [修改] 在卡片标题区域添加坐标显示 -->
            <template #extra>
              <span v-if="originalImageSize">
                原始分辨率: ({{ originalImageSize.width }}, {{ originalImageSize.height }})
              </span>
              <span v-if="mouseCoordinates">
                坐标: ({{ mouseCoordinates.x }}, {{ mouseCoordinates.y }})
              </span>
            </template>
            <!--
              [核心修改]
              将事件监听器从 a-image 移到这个 div 上。
              这个 div 是我们自己控制的，事件绑定更可靠。
            -->
            <div
                class="screenshot-wrapper"
                ref="screenshotWrapperRef"
            >
<!--              <a-image-->
<!--                  :src="screenshot"-->
<!--                  :preview="false"-->
<!--                  class="screenshot-img"-->
<!--                  v-if="screenshot"-->
<!--              />-->
              <!-- [核心修改] 将 a-image 替换为 canvas -->
              <canvas
                  ref="canvasRef"
                  class="screenshot-img"
                  v-show="screenshot"
                  @mousemove="handleScreenshotMousemove"
                  @mouseleave="handleScreenshotMouseleave"
                  @click="handleCanvasClick"
              ></canvas>

              <a-empty v-if="!screenshot" description="请先获取截图" />
            </div>
          </a-card>
        </a-col>

        <!-- 标注编辑列 -->
        <a-col :span="12">
          <a-card title="模型推理结果">
            <a-skeleton :loading="modelLoading">
              <!-- 如果没有模型响应，显示提示信息 -->
              <a-empty v-if="!screenshot" description="等待图片加载..." />

              <!-- [修改] 使用结构化表单替代文本域 -->
              <a-form v-else layout="vertical" :model="actionForm" class="annotation-editor">
                <!-- Action Type 下拉框 -->
                <a-form-item label="操作类型 (Action)">
                  <a-select
                      v-model:value="actionForm.action"
                      :options="actionTypes"
                      placeholder="请选择操作类型"
                  />
                </a-form-item>

                <!-- 条件渲染: tap 或 long_press -->
                <a-space v-if="actionForm.action === 'tap' || actionForm.action === 'long_press'">
                  <a-form-item label="坐标 X">
                    <a-input-number v-model:value="actionForm.x" placeholder="X" />
                  </a-form-item>
                  <a-form-item label="坐标 Y">
                    <a-input-number v-model:value="actionForm.y" placeholder="Y" />
                  </a-form-item>
                </a-space>

                <!-- 条件渲染: text -->
                <a-form-item label="输入内容" v-if="actionForm.action === 'text'">
                  <a-input v-model:value="actionForm.value" placeholder="请输入文本" />
                </a-form-item>

                <!-- 条件渲染: need_feedback -->
                <a-form-item label="输入需要反馈给用户的内容" v-if="actionForm.action === 'need_feedback'">
                  <a-input v-model:value="actionForm.value" placeholder="请输入文本" />
                </a-form-item>

                <!-- 条件渲染: swipe -->
                <template v-if="actionForm.action === 'swipe'">
                  <a-space>
                    <a-form-item label="起始坐标 X">
                      <a-input-number v-model:value="actionForm.x" placeholder="X" />
                    </a-form-item>
                    <a-form-item label="起始坐标 Y">
                      <a-input-number v-model:value="actionForm.y" placeholder="Y" />
                    </a-form-item>
                  </a-space>
                  <a-space>
                    <a-form-item label="滑动方向">
                      <a-select
                          v-model:value="actionForm.direction"
                          :options="swipeDirections"
                          style="width: 120px"
                      />
                    </a-form-item>
                    <a-form-item label="滑动距离">
                      <a-select
                          v-model:value="actionForm.distance"
                          :options="swipeDistances"
                          style="width: 120px"
                      />
                    </a-form-item>
                  </a-space>
                </template>
                <!-- 条件渲染: swipe_two_points -->
                <template v-if="actionForm.action === 'swipe_two_points'">
                  <a-space>
                    <a-form-item label="起始坐标 X">
                      <a-input-number v-model:value="actionForm.x" placeholder="X" />
                    </a-form-item>
                    <a-form-item label="起始坐标 Y">
                      <a-input-number v-model:value="actionForm.y" placeholder="Y" />
                    </a-form-item>
                  </a-space>
                  <a-space>
                    <a-form-item label="终止坐标 X">
                      <a-input-number v-model:value="actionForm.x_end" placeholder="X" />
                    </a-form-item>
                    <a-form-item label="终止坐标 Y">
                      <a-input-number v-model:value="actionForm.y_end" placeholder="Y" />
                    </a-form-item>
                  </a-space>
                </template>
                <!-- wait 和 FINISH 不需要额外输入框 -->

                <a-divider />
                <div class="action-bar">
                  <a-button @click="handlePreview" :disabled="!actionForm.action">预览操作</a-button>
                  <a-button
                      type="primary"
                      @click="handleSaveAnnotation"
                      :disabled="!actionForm.action"
                  >
                    保存并执行
                  </a-button>
                  <a-button @click="handleReset" :disabled="!rawModelResponse">重置修改</a-button>
                </div>
              </a-form>
            </a-skeleton>
          </a-card>
        </a-col>
      </a-row>
    </a-layout-content>
  </a-layout>
</template>

<style scoped>
.container {
  height: 100vh;
  padding: 20px;
  margin: 0; /* 居中 */
  width: 100vw;
}

.content {
  padding: 24px;
  background: #fff;
}

.screenshot-wrapper {
  flex: 1;
  overflow: hidden;
  border: 1px dashed #d9d9d9;
  position: relative;
  background-color: #fafafa;

  /* --- 新增样式：用于垂直和水平居中canvas元素 --- */
  display: flex;
  justify-content: center; /* 水平居中 */
  align-items: center;     /* 垂直居中 */
}

/* [修改] 样式现在应用到 canvas 上
设置图片显示高度为固定值
*/
.screenshot-img {
  display: block;
  height: 70vh;
}

.annotation-editor {
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.action-bar {
  display: flex;
  gap: 12px;
  justify-content: flex-end;
}
</style>