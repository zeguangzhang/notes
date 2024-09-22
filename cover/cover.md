# 封面图的整体演进过程
## 合作方数据部门研发 pdf
## 合作方数据部门研发 coverImage
- 借用他们的无头浏览器壳子，但是需要让他们监控headless可以关闭的信号，需要前端在headless上截封面后发送一个请求给服务端，进行封面存储，
这个请求成功结束后，给予设置一个window.{设置的一个变量}为true。无头浏览器监控到这个变量为true后关闭浏览器；
- 再次去找合作方大头兵支持我们还有pptx下载： pptx和coverImage是一个逻辑，但是pptx发现不行，去找合作方.而且pptx那个耗时更长，需要等前端把图坐标画出来再传给后端接口，后端接口再调用算法。合作方认为这不是他们的范围，嵌入了业务逻辑，不打算做支持，包括这个coverImage,以后都需要我们自己搞。
- 所以去找他们领导说这件事，然后拉会叫上多方leader讨论。最终决定：pdf他们还是支持，其他我们用自己的node搞，封面图先让他们临时支持，我们这边稳定后切过来
```azure
这里想说的是真不如自己研发，因为跨部门沟通成本太高了，远比自己开发耗费时间更长。
```
## 自己研发
- 重新部署node服务
- pptx调研发送不通问题，因为权限问题，看不到机器真实情况，应该是前阵子公司回收不用的资源，给缩小了，现在就是1核1g,不够pptx用的问题，升级资源到4h4g后解决导出问题
- pptx导出图片能力不行，图基本都没出来，给前端看，前端又说我们的问题，于是我就在导出pptx的页面上截一张全页图。发现在测试服务上没任何问题，之后又给到前端，前端说产品说这个需求先不上了。 
- coverImage 把截图能力放到我们自己后端，然后发送封面图请求，这当时还调试了大半天（node发送图片的请求）。
```nodejs
# 但是主要卡点就是图片还没截图完就走到了下一步，导致错误。需要检测下图片可用后再去进行图片操作
async function screenOnePage(requestId, pptId, elementHandles, pageNum, uploadScreenUrl) {
    const elementHandle = elementHandles[pageNum];
    // 下面代码调试用
    // let screenPath = `image/screenshot-${pageNum}.png`
    // await elementHandle.screenshot({path: screenPath});
    // // 检查文件是否存在并且可读取
    // try {
    //     await fs.promises.access(screenPath, fs.constants.F_OK | fs.constants.R_OK);
    // } catch (err) {
    //     console.error(`File not available for upload: ${screenPath}`);
    //     return;
    // }
    //
    // // 检查截图文件是否存在
    // if (!fs.existsSync(screenPath)) {
    //     console.error(`File does not exist: ${screenPath}`);
    //     return;
    // }
    const screenshotBuffer = await elementHandle.screenshot();
    // 创建一个可读流
    const readableStream = new Readable();
    readableStream.push(screenshotBuffer);
    readableStream.push(null);

    const formData = new FormData();
    // formData.append('file', screenshotBuffer);
    formData.append('file', readableStream, {
        filename: 'screenshot.png',
        contentType: 'image/png'
    });
    formData.append('ppt_id', pptId);
    formData.append('image_type', 0);
    formData.append('pageNum', 1);// todo 根据请求参数获取,接收list,传0是获取所有
    axios.post(uploadScreenUrl, formData, {
        headers: {
            ...formData.getHeaders()
        }
    })
        .then(response => {
            log.info(` 上传成功`, response.data);
        })
        .catch(error => {
            log.error(` 上传失败`, error);
        });
}
```
- coverImage 直接oss存储，不用发送了，每次过来传给我osskey
- screenlist 也开始做，每页截图放到每页的oss key里，除第一次全部截图外，之后都是更新了哪页截哪页
- coverImage表情，字体问题
- coverImage保证等宽，宽高比要在16/9和3/4之间，，宽高要都大于400像素, 否则就对高度就行填充空白或者裁剪
```azure
卡点：版本问题
- 这用node的jimp包，但是调试时发现下面这句话总报错，说不支持的read方法
const originalImage = await Jimp.read(screenshotBuffer); // screenshotBuffer是图片的内存二进制
- 我去看Jimp最新1.6.0的包确实这么用的，只不过是读的本地文件，读本地文件发现也没问题，之后我就下调到几个版本，发现还是不行，但是我在下调版本中发现在安装时提示最低需要的node版本都比node18高。
- 我接着去chatgpt一下node18需要用jimp哪个版本合适，结果是："jimp": "^0.16.2",版本，我看0.16.2有两个版本，我就故意上调了一个版本0.16.3，发现可用了。
```
- coverImage要从内容的图片中抽取，等宽，宽高比要在16/9和3/4之间，宽高要都大于400像素
```azure
- 上级要求正则抽取改为结构化抽取（含递归），个人觉得正则应该更高效；
- 图片走过滤，符合的图片留下返回
- 因为每次看图片是否符合，我要先下载下来，这就涉及到多图片下载会导致接口缓慢，所以我限制了10张图片，之后做过滤
- 图片path跟我们的screenlist的图片的域名不同，因为我们存储的是相对路径，所以我们需要把他们的图片下载下来重新put到我们的oss上。但是因为时间紧急，只能先调用公共的一个方法做替换，但是这样导致这又下载了一次图片。
- 做重复下载优化：在自己下载后put到oss上
```





