<!DOCTYPE html>
<html lang="zh-CN">
<head>
<meta charset="UTF-8">
<meta content="width=device-width,initial-scale=1.0,maximum-scale=1.0,user-scalable=0" name="viewport">
<title>Tiele</title>
<style>
    html, body {
      margin: 0; 
      padding: 0; 
      width: 100%; 
      height: 100%; 
      overflow: hidden; 
    }
    .content { 
      position: fixed; 
      width: 100%; 
      height: 100%; 
      left: 0; 
      top: 0; 
      border: none; 
      opacity: 0;
      transform: translateY(10px);
      transition: opacity 0.5s cubic-bezier(0.4,0,0.2,1), transform 0.5s cubic-bezier(0.4,0,0.2,1);
    }
    .error-container {
      display: none;
      flex-direction: column;
      justify-content: center;
      align-items: center;
      height: 100vh;
      text-align: center;
      padding: 0 20px;
    }
    .error-main {
      font-size: 22px;
      color: #333;
      margin-bottom: 16px;
      font-weight: 500;
    }
    .error-contact {
      font-size: 18px;
      color: #666;
      opacity: 0.9;
    }
    .loader {
      position: fixed;
      top: 0;
      left: 0;
      width: 100%;
      height: 100%;
      display: flex;
      justify-content: center;
      align-items: center;
      z-index: 9999;
      opacity: 1;
      transition: opacity 0.3s ease-out, transform 0.3s ease-out;
      background: #fff;
    }
    .spinner {
      width: 50px;
      height: 50px;
      border: 4px solid rgba(0,122,255,0.15);
      border-radius: 50%;
      border-top-color: #007AFF;
      border-bottom-color: #007AFF;
      animation: spin 1.2s cubic-bezier(0.6,0,0.4,1) infinite;
    }
    @keyframes spin {
      0% { transform: rotate(0deg); }
      100% { transform: rotate(360deg); }
    }
    .loader-hide {
      opacity: 0;
      transform: scale(0.9);
      pointer-events: none;
    }
    .content-show {
      opacity: 1;
      transform: translateY(0);
    }
</style>
</head>
<body>
<div class="loader" id="loader">
  <div class="spinner"></div>
</div>
<div class="error-container" id="errorContainer">
  <div class="error-main" id="errorMain"></div>
  <div class="error-contact">@AGE0118</div>
</div>
  
<script>
    const hideLoader = () => {
      const loader = document.getElementById('loader');
      loader.classList.add('loader-hide');
      setTimeout(() => {
        loader.style.display = 'none';
      }, 300);
    };

    const showError = (mainText) => {
      document.getElementById('errorMain').textContent = mainText;
      document.getElementById('errorContainer').style.display = 'flex';
    };

    // 优化：增加白名单请求重试机制（最多3次）
    const fetchWhiteList = async (retry = 3) => {
      try {
        const response = await fetch('http://youyu.es/Yao/Api.php'); 
        if (!response.ok) throw new Error(`HTTP错误: ${response.status}`);
        
        const whitelistStr = response.headers.get('X-Whitelist-Domains') || '';
        const whiteList = whitelistStr.split(',').filter(domain => domain.trim() !== '');
        if (whiteList.length === 0) {
          throw new Error('白名单为空，请联系管理员配置');
        }
        return whiteList;
      } catch (error) {
        console.error(`白名单加载失败（剩余重试次数：${retry - 1}）:`, error.message);
        if (retry > 1) {
          await new Promise(resolve => setTimeout(resolve, 1000)); // 间隔1秒重试
          return fetchWhiteList(retry - 1);
        }
        showError('白名单加载失败，请稍后重试');
        hideLoader();
        return []; 
      }
    };

    const fetchTitleByProxy = async (url) => {
      try {
        const proxyUrl = `http://dingun.sbs/Yao/proxy.php?url=${encodeURIComponent(url)}`;
        const response = await fetch(proxyUrl);
        if (!response.ok) throw new Error(`HTTP错误: ${response.status}`);
        
        const data = await response.json();
        return data.title || '官方网页，安全无患';
      } catch (error) {
        console.warn('通过代理获取标题失败:', error.message);
        return new URL(url).hostname;
      }
    };

    // 优化：更严谨的域名匹配逻辑（支持主域名和子域名）
    const isDomainAllowed = (domain, whiteList) => {
      // 统一转为小写，去除前缀www.（避免www.example.com和example.com不匹配）
      const normalizedDomain = domain.toLowerCase().replace(/^www\./, '');
      return whiteList.some(allowed => {
        const normalizedAllowed = allowed.toLowerCase().replace(/^www\./, '');
        // 匹配规则：完全相等 或 子域名（如allowed为example.com，匹配a.example.com）
        return normalizedDomain === normalizedAllowed || 
               normalizedDomain.endsWith(`.${normalizedAllowed}`);
      });
    };

    document.addEventListener('DOMContentLoaded', async () => { 
      const WHITE_LIST = await fetchWhiteList();
      if (WHITE_LIST.length === 0) return; // 白名单为空时终止

      const urlParams = new URLSearchParams(window.location.search);
      const encodedParam = urlParams.get('p');
      
      if (!encodedParam) {
        showError('缺少必要参数');
        hideLoader();
        return;
      }
      
      try {
        // 优化：增加base64解码容错
        let tureurl;
        try {
          tureurl = decodeURIComponent(atob(encodedParam));
        } catch (e) {
          throw new Error('参数解码失败（格式错误）');
        }
        
        if (!tureurl.startsWith('http://') && !tureurl.startsWith('https://')) {
          throw new Error('必须以http或https开头');
        }
        
        const url = new URL(tureurl);
        const domain = url.hostname;
        
        if (!isDomainAllowed(domain, WHITE_LIST)) {
          showError(`域名未授权${domain}`); // 显示具体域名，便于排查
          hideLoader();
          return;
        }
        
        const pageTitle = await fetchTitleByProxy(tureurl);
        document.title = pageTitle;
        
        const iframe = document.createElement('iframe');
        iframe.className = 'content';
        iframe.src = tureurl;
        // 优化：增加安全限制
        iframe.sandbox = 'allow-same-origin allow-scripts allow-popups allow-forms';
        iframe.allow = 'same-origin';
        document.body.appendChild(iframe);
        
        // 优化：处理iframe加载失败
        iframe.onerror = () => {
          showError(`页面加载失败：${domain}`);
          hideLoader();
        };
        
        iframe.onload = () => {
          hideLoader();
          iframe.classList.add('content-show');
        };
        
        if (iframe) bindMouseWheel(iframe);
      } catch (error) {
        console.error('页面加载失败:', error.message);
        showError(error.message || '页面加载失败，请检查URL');
        hideLoader();
      }
    });

    const isFirefox = navigator.userAgent.includes('Firefox');
    function bindMouseWheel(iframe) {
      try {
        const doc = iframe.contentWindow.document;
        const handleWheel = (e) => {
          e.preventDefault();
          const delta = isFirefox ? e.detail * -50 : e.wheelDelta;
          doc.documentElement.scrollTop += delta;
        };
        doc.addEventListener(isFirefox ? 'DOMMouseScroll' : 'wheel', handleWheel);
      } catch (e) {
        console.warn('跨域限制:', e); 
      }
    }
  </script>
</body>
</html>
