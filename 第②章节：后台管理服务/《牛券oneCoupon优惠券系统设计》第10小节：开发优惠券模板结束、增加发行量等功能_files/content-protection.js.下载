function filterRLOLRO(str) {
  return str.replace(/[\u202e\u202d]/g, '');
}
function setWatermarkText(user_id) {
  var options = {
    container: document.querySelector('.js_watermark'),
    content: filterRLOLRO(user_id),
    dx: 80, // x轴方向上 文字container的间距
    fontWidth: 0,
    rotate: -20,
    fontSize: '14',
    fontFamily: 'PingFang SC',
    fillStyle: 'rgba(0,0,0,0.04)',
  };
  generateWatermark(options);
}
var deg = Math.PI / 180;
function getRatio(ctx) {
  var devicePixelRatio = window.devicePixelRatio || 1;
  var backingStoreRatio = ctx.webkitBackingStorePixelRatio || 1;
  return devicePixelRatio / backingStoreRatio;
}
function generateWatermarkItem(options) {
  var canvas = document.createElement('canvas');
  var ctx = canvas.getContext('2d');
  var ratio = getRatio(ctx);

  var fontSize = Number(options.fontSize) * ratio;
  ctx.font = fontSize + 'px ' + options.fontFamily;
  var fontWidth = ctx.measureText(options.content).width;

  var height =
    fontWidth * Math.sin(deg * Math.abs(options.rotate)) + fontSize * 1.5;
  var width =
    fontWidth * Math.cos(deg * Math.abs(options.rotate)) + fontSize;

  options.height = height;
  options.width = width;
  options.fontWidth = fontWidth;

  canvas.setAttribute('width', width + 'px');
  canvas.setAttribute('height', height + 'px');

  ctx.translate(0, height);
  ctx.rotate((Math.PI / 180) * options.rotate);
  ctx.font = fontSize + 'px ' + options.fontFamily;
  ctx.fillStyle = options.fillStyle;
  ctx.fillText(options.content, fontSize / 2, -fontSize / 2);

  return canvas;
}
function generateWatermark(options) {
  var canvas = document.createElement('canvas');
  var ctx = canvas.getContext('2d'); 
  var cw = generateWatermarkItem(options);
  var yGutter = options.height * 0.5

  var ratio = getRatio(ctx);
  var unitWidth = options.width + options.dx;
  var unitHeight = options.height + yGutter;
  

  console.log(ratio)

  var yCount = 2 // line
  var containerWidth = options.container.offsetWidth;
  var containerHeight = ((unitHeight) / ratio) * yCount ;
  var xCount = Math.ceil(containerWidth / unitWidth) * ratio;

  canvas.setAttribute('width', containerWidth * ratio + 'px');
  canvas.setAttribute('height', containerHeight * ratio + 'px');

  for (var i = 0; i <= yCount + 1; i++) {
    // 竖排
    for (var j = 0; j <= xCount; j++) {
      // 横排
      var offsetX = unitWidth * j;
      var offsetY = unitHeight * i;
      if (i % 2 === 0) {
        offsetX -= unitWidth * 0.3;
      }
      ctx.drawImage(cw, offsetX, offsetY);
    }
  }

  var base64Url = canvas.toDataURL();

  options.container.style.cssText += `
    ;background-repeat: repeat; 
    background-image: url( ${base64Url} );
    background-size: ${containerWidth}px ${containerHeight}px
  `;
}
function setDisableCopy() {
  document.body.classList.add('js-disable-copy');
  document.addEventListener('copy', disableCopyCallback);
  document.addEventListener('contextmenu', disableCopyCallback);
}
function disableCopyCallback(event) {
  var text = '本星球已开启内容保护，⽆法复制主题内容';
  if (isAppTemplate) {
    zsxq_js.showToast(text);
  } else {
    message(text);
  }
  event.preventDefault();
  return false;
}

function getValue(name) {
  var val = document.querySelector('[name="' + name + '"]').value;
  if (val.startsWith('###') && val.endsWith('###')) {
    return ''
  } else {
    return val
  }
}
var $data = {
  group_allow_copy: getValue('group_allow_copy') === 'true', // String! 'true' | 'false'
  group_enable_watermark: getValue('group_enable_watermark') === 'true', // String! 'true' | 'false'
  member_id: getValue('member_id'),
  member_name: getValue('member_name'),
  member_role: getValue('member_role'), // owner | admin | partener | guest | other
};
var ALLOW_COPY_ROLES = ['owner', 'admin', 'partener'];

if ($data.group_enable_watermark && $data.member_name && $data.member_id) {
  setWatermarkText($data.member_name + ' ' + $data.member_id);
}

if (
  !$data.group_allow_copy &&
  ALLOW_COPY_ROLES.indexOf($data.member_role) === -1
) {
  setDisableCopy();
}
