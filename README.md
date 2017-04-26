# facelocation
臉部定位


Raspberry Pi 3 人臉定位結合PiCamera鏡頭與步進馬達轉向
作者:林大為
研究動機
  為了學習raspberry 與產出相關作品，研究各種不一樣的東西，是我的興趣，透過這次實作，希望藉由這次實驗，可以對於影像辨識可以有進一步的學習，並且透過操作RaspberryPi 更加熟練的操作Linux系統。
硬體
Raspberry Pi3	*1
Raspberry 專用鏡頭		*1
軟體與編輯器
RaspBian 4.4
Eclipse  Mars.2 Release (4.5.2)
Java jdk8
Putty(+pscp)
NetBean + Pi4j



摘要
  本次實驗著重於，人臉快速定位，除了在移動時進行同步運算，運用步進馬達旋轉鏡頭來追蹤臉部，實驗屬於單一最佳化人臉追蹤，會針對最符合臉部的特徵做追蹤。
關鍵詞：偵測膚色、HSV、RGB、人臉特徵、人臉辨識、RaspBerryPi3。

人臉偵測
  我認為最重要的部分就是膚色判斷，然而傳統的膚色判斷RGB，會因為光線問題造成失準，所以我採用HSV 搭配RGB來框定膚色範圍，再用臉型判斷，找出最適合的臉部特徵區塊，大部分區塊選擇都會採用Connected Componemt，但是經過我測試後發現，如果在光源不均的狀態下，所造成的黑影，與臉上配件，例如眼鏡，都會造成Connected Componemt切割失敗，因此我另外增加一組自創Cross nose tracking判斷演算法，先針對人臉做妥元修復，移除頭髮雜訊等相關過濾，提高臉部追蹤的速度與減少判斷誤差。





流程圖
 


 
實驗步驟

Raspberry Pi基本安裝
  我這邊採用的是RaspBian 4.4 版本，下載完畢後，使用Fedora Installer 將鏡像檔案安裝到Raspberry Pi 上的SD卡，安裝完畢後必須注意有沒有空間無法被格式找到的問題，可以使用Win32DiskImager 這個工具來回復原有的記憶卡空間。
  開機後記得”sudo raspi-config”做相關的設定，網路我是採用Router local net work，所以請記得設定固定DHCP IP ，目前開發是透過window OS 所以下載putty做連結(PSCP做檔案傳輸器)。

Raspberry Pi Java Embedded安裝與說明
製作 java Embedded:
bin\jrecreate.bat --profile compact1 --dest myjre\mycompact1\ bin\jrecreate.bat --profile compact2 --dest myjre\mycompact2\ bin\jrecreate.bat --profile compact3 --dest myjre\mycompact3\
  設定完JMC(Java Mission control)，就可以透過SSH 來監控Java 的Activity information，監測java 個執行續的狀態。

Pi4j 的使用
為了實現分層與快速開發，我採用Pi4J api，透過api 與下層的C做溝通，透過這樣模擬出android 五層的架構，因此這次開發多屬於在相對Application layer 與Application Framework layer位置。

Raspberry Pi Pi4j 接腳圖
 



定位流程分析與詳解
這邊開始，我會一個個分拆程式碼，雖然不是非常詳細，希望能夠在以後，幫助我快速回復記憶。
RGB 轉 HSV
The R,G,B values are divided by 255 to change the range from 0..255 to 0..1:
R' = R/255
G' = G/255
B' = B/255
Cmax = max(R', G', B')
Cmin = min(R', G', B')
Δ = Cmax - Cmin
 
Hue calculation:
 
 
Saturation calculation:
 
 
Value calculation:
V = Cmax
實際程式碼: https://github.com/DavidDaWei/facelocation.git

HSV 轉 RGB
When 0 ≤ H < 360, 0 ≤ S ≤ 1 and 0 ≤ V ≤ 1:
C = V × S
X = C × (1 - |(H / 60°) mod 2 - 1|)
m = V - C
 
(R,G,B) = ((R'+m)×255, (G'+m)×255, (B'+m)×255)

實際程式碼: https://github.com/DavidDaWei/facelocation.git




 
MedianFilter(中值濾波)
有一個補充，中值濾波跟平滑濾波(補1)是非常有趣的兩個演算法。
// 中值濾波
	private BufferedImage MedianFilter(BufferedImage image) {
		int arr[][] = getImageArray(image, image.getWidth(), image.getHeight());
		int w = image.getWidth();
		int h = image.getHeight();
		for (int x = 1; x < image.getWidth(); x++) {
			for (int y = 1; y < image.getHeight(); y++) {
				// RGB 中值濾波
				int[] ChallengeArr = new int[9];
				ChallengeArr[4] = arr[x][y];// 中間
				ChallengeArr[1] = arr[x - 1][y - 1];// 左上
				ChallengeArr[2] = arr[x][y - 1];// 上
				if (x + 1 < w)
					ChallengeArr[3] = arr[x + 1][y - 1];// 右上
				if (x + 1 < w)
					ChallengeArr[0] = arr[x + 1][y];// 右
				if (x + 1 < w && y + 1 < h)
					ChallengeArr[5] = arr[x + 1][y + 1];// 右下
				if (y + 1 < h)
					ChallengeArr[6] = arr[x][y + 1];// 下
				if (y + 1 < h)
					ChallengeArr[7] = arr[x - 1][y + 1];// 左下
				ChallengeArr[8] = arr[x - 1][y];// 左
				Arrays.sort(ChallengeArr);
				image.setRGB(x, y, ChallengeArr[4]);
			}
		}
		return image;
	}
 
Binarization(二質化)
  二值化我採用動態閥值抓取，透過HSV 我們可以發現每個像素獨立亮度，我算出整張圖的平均亮度，再透過實驗，找出最佳的量化梯度表，並且對應到最佳的閥值，其中二質化前必須將像素RGB灰階化(補2)。

	/** 第一線運算公式: 二值化圖片 (先做灰階) **/
	public BufferedImage PicBinary(BufferedImage sourceImg, int Width, int Height, int FZ) {
		int arr[][] = getImageGrayArray(sourceImg, Width, Height);// 灰階陣列獲得
		BufferedImage image = new BufferedImage(Width, Height, BufferedImage.TYPE_BYTE_BINARY);
		for (int i = 0; i < Width; i++) {
			for (int j = 0; j < Height; j++) {
				if (arr[i][j] >= FZ) {
					image.setRGB(i, j, Color.white.getRGB());
				} else {
					image.setRGB(i, j, Color.black.getRGB());
				}
			}
		}
		return image;
	}
 
PhaianErosion(方陣侵蝕)
  這是一個方陣型的圖片侵蝕演算法，通常會搭配膨脹演算(Dilate)法來達到閉運算(Closed operation)或是開運算(Opened operation)，大事我發現，在複雜背景下的臉部偵測，不適合用開/閉運算，最好的方法還是透過膚色先切出邊界、濾除雜訊，最後在讓臉部與周邊光線分離，分離的部分我就用侵蝕來達成。
  這邊大致上用到 3*3 5*5 7*7 的方陣，用大的方陣效果用明顯，但是錯誤率也相對提高。
程式碼: https://github.com/DavidDaWei/facelocation.git
 
Connected Componment
連通標記演算法，這是一個被廣泛使用在各論文中，最陽春的區塊切割法，在許多論文中(台灣的)，幾乎都忽視一個嚴重的問題 “眼鏡”，當然也有特別用機器學習的方式，將眼鏡或其他配飾濾除，但是本篇的重心就是，快速找到臉部，所以我思考許久，決定不使用這個演算法。
程式碼: https://github.com/DavidDaWei/facelocation.git

測試十字人臉定位 
透過 Connected Componment 可以將臉部區塊區分出來，但是我發現因為眼鏡的關係，上臉部(額頭、眼睛) 都有機會被誤判刪除，所以我觀察我二值化的圖片，發現雜訊部分與臉部，最大的差別，就是變化率，臉部經過上述MedianFilter PhalanxErosion 等等步驟後，幾乎剩下眼睛與嘴巴，還有鼻孔，因此色塊變化率非常低，我透過寬高掃描，記錄下每個軸向的色塊變化，黑轉白+1 還有白轉黑+1 計算出每軸的變化數 n
再來透過白區塊面積掃描算出軸的白像素 t
Δp = t – n
簡單的完成十字定位，經過測試，準確度高達 90% 而且速度是控制在平均 50 ms 以下。
 
總結
總耗時: 15天
參考文獻
曾婉菁/中央印製廠技研室管理師

Daniel B. Wright a,*, Benjamin Sladden b-
An own gender bias and the importance of hair in face recognition

Harry Wechsler ,P. Jonathon Phillips ,Vicki Bruce ,
Fran~oise Fogelman Soulie ,Thomas S. Huang-  
Face Recognition From Theory to Applications

How effective are landmarks and their geometry for face recognition?
J. Shi a,*, A. Samal a,*, D. Marx b

Discriminant Analysis of Principal Components for Face Recognition
W. Zhao R. Chellappa A. Krishnaswamy

Illumination Normalization for Face Recognition and Uneven Background Correction Using Total Variation Based Image Models
Terrence Chen1 , Wotao Yin2, Xiang Sean Zhou3, Dorin Comaniciu3, and Thomas S. Huang1 1University of Illinois at Urbana Champaign, 2Columbia University, 3Siemens Corporate Research {1tchen5, huang}

