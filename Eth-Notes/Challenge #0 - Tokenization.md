# Challenge #0 - Tokenization

### 源码解析：

Tokenization挑战最核心的就是NFT元数据的上传与获取CID，原代码里在nftsMetadata.ts里设置好了5个NFT元数据，所以我们只需要把这些数据打包上传到IPFS上，然后从IPFS上获取上传的数据url显示到本地。其中：

**nextjs/app/api/ipfs**文件夹下是路由，其中**add**是上传NFT到IPFS的路由，**get-metadata**是从IPFS上获取NFT的元数据路由；

**nextjs/app/myNFTs/_components**文件夹下是NFT的一个代码组件，用于设计NFT的组成结构；

**nextjs/app/myNFTs/page.tsx**是NFT展示页面的前端代码组件，用于展示所拥有的NFT；

**nextjs/app/utils/tokenization**是工具组件，用于NFT上传至IPFS和获取元数据的代码实现逻辑；

以上文件大致构成NFT的整个铸造逻辑。

### **问题：**

但是从源代码中并不能从前端页面完整显示NFT，原因是IPFS公有链接有问题，不能上传和读取NFT数据，所以我们可以换一个，这里通过Pinata完成NFT铸造的一整个流程。

### 过程：

1、进入官网[https://app.pinata.cloud/](https://app.pinata.cloud/) 注册生成一个自己的API密钥，记住生成的API密钥信息。这里我们需要用上其中的访问令牌和网关地址，然后我们将这些信息放在新建的环境变量里，从而在使用时通过环境变量来访问pinata，这样可以防止这些关键隐私信息泄露。

2、修改上传路由将NFT元数据转至pinata，以下是部分代码逻辑：

```tsx
const PINATA_JWT = process.env.PINATA_JWT;
const PINATA_PIN_JSON_ENDPOINT = "https://api.pinata.cloud/pinning/pinJSONToIPFS"; 
export async function POST(request: Request) {
  try {
    if (!PINATA_JWT) {
      throw new Error("Missing env var: PINATA_JWT");
    }
    const body = await request.json();
  
    const pinataRes = await fetch(PINATA_PIN_JSON_ENDPOINT, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${PINATA_JWT}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify({
        pinataOptions: { cidVersion: 1 },
        pinataMetadata: { name: "metadata.json" },
        pinataContent: body,
      }),
    });
    if (!pinataRes.ok) {
      const text = await pinataRes.text();
      throw new Error(`Pinata upload failed: ${pinataRes.status}${text}`);
    }
    const data: {
      IpfsHash: string;
      PinSize: number;
      Timestamp: string;
      isDuplicate?: boolean;
    } = await pinataRes.json();

    return Response.json({
      path: data.IpfsHash,
      cid: data.IpfsHash,
      size: data.PinSize,
      timestamp: data.Timestamp,
      isDuplicate: data.isDuplicate ?? false,
    });
```

3、解析上传IPFS上NFT元数据的CID，以下是部分代码逻辑：

```tsx
export async function getNFTMetadataFromIPFS(ipfsHash: string) {
  const cid = ipfsHash
    .replace(/^ipfs:\/\//, "")
    .replace(/^ipfs\//, "")
    .split("/ipfs/").pop()!
    .split("?")[0]
    .split("#")[0];
  const url = `https://gateway.pinata.cloud/ipfs/${cid}`;

  const r = await fetch(url, { cache: "no-store" });

  if (!r.ok) return undefined;
  return await r.json();
}
```

这样我们就可以完成整个NFT的铸造流程，前端页面也能够显示铸造的NFT。