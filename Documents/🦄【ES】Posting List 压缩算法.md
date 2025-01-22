解决 Posting List 需要存储数据过多导致的空间浪费问题。
## 索引帧（Frame Of Reference）
适用于分散紧密的情况，存储的是前后两个数字的差值。
![[Frame Of Reference 算法.png|1000]]

## 咆哮位图（Roaring Bitmap）
适用于稀疏分散的情况，存储的是对某个数字做除法和取余之后的坐标值。
![[咆哮位图压缩算法.png]]