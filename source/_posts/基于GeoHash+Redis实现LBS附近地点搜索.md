title: 基于GeoHash+Redis实现LBS附近地点搜索
toc: true
date: 2015-10-21 15:55:27
categories:
    - geohash
tags:
    - geohash
    - lbs
---

![GeoHash](/img/geohash.png)

###需求

随着移动业务的不断创新，基于位置服务的需求逐渐增多。比如美团搜索附近的美食，嘀嘀打车搜搜附近的司机等。支撑这一业务最主要的就是LBS(基于位置的服务)，特别是嘀嘀打车的业务尤为突出，司机与乘客不断更新的坐标地址，从一大堆无关的坐标地址内找出一定范围内所有的司机，这无疑对服务器承载力带来巨大的挑战。

先比较一下不同的解决方案。最后讲解一下如何使用52位的GeoHash集成Redis实现此需求。

###解决方案

1. 物理计算，正对某个中心点，计算所有周边点，再根据搜索范围刷选。
2. 基于MySQL，可选择物理计算/GeoHash/MySQL空间索引。
3. 基于MongoDB，依赖MongoDB的空间搜索算法搜索。
4. 自行开发搜索算法并存入Redis。

<!-- more -->

###方案解析

#####物理计算

在我们第一次遇到这类问题，头脑里面最先浮现的解决方案，但是我们的潜意思告诉我们这肯定是不行的。如果搜索的受众量很大，光是计算距离就要消耗大量的时间。

#####基于MySQL

存入MySQL实现上有三种方式：

1. 根据纬度和精度每增加0.01度增加1000m为算法，检索出候选坐标，再根据搜索范围刷选。距离搜索3公里以内所有目标，伪代码：

    ```java
    double kilometer = 3.0;
    
    double precision = 0.01 * kilometer;
    
    //SQL 限定条件 
    SELECT * FROM driver_coordinate
    WHERE latitude > latitude - precision && 
          latitude < latitude + precision &&
          longitude > longitude - precisionRadius && 
          longitude < longitude + precisionRadius;
    ```
2. 给每一个坐标计算一个[GeoHash](https://en.wikipedia.org/wiki/Geohash)值，业界大部分的使用的是基于Base32编码的12位GeoHash字符串。但是这样精度不太容易控制。我们将要讨论的是52位的整形值来表示GeoHash。52位的GeoHash最小能精确到0.6米，足够满足绝大部分需求(52位GeoHash算法见下文)。
    
3. MySQL空间存储，MySQL本省就支持空间搜索。
    
    ```
    GLength(LineStringFromWKB(LineString(point1, point2)))
    ```
    
但是由于MySQL的限制，查询效率低，可伸缩性较差。
    
    
#####基于MongoDB

MonggoDB原生也支持地理位置索引，支持位置查询及排序。

命令如:

```
db.runCommand({ geoNear: "places", near: [30.545162, 104.062018], num:1000 })
```

MongoDB支持地理位置查询，可伸缩性较好，配置集群简单，集成方便快，成本最低。3.0版本之后增加了Collection锁，性能大大提升。业界也有不少公司在使用，比如美团、街旁、全球最大的LBS应用[Foursquare](https://foursquare.com/)用的MongoDB。

#####自行开发搜索算法并存入Redis

自行开发算法精度与扩展方式容易控制，对第三方没有依赖方便升级与替换。

### 具体实现

1. 坐标转换成52bit二进制编码值。

    * 52bit精度在0.6m足够满足搜索范围需求。
    * 算法，根据经纬度计算GeoHash二进制编码：
    
        * 首先将纬度范围(-90, 90)平分成两个区间(-90, 0)、(0, 90)， 如果目标纬度位于前一个区间，则编码为0，否则编码为1。由于39.92324属于(0, 90)，所以取编码为1。然后再将(0, 90)分成 (0, 45), (45, 90)两个区间，而39.92324位于(0, 45)，所以编码为0。以此类推，直到精度符合要求为止，得到纬度编码为1011 1000 1100 0111 1001。
        * 经度也用同样的算法，对(-180, 180)依次细分，得到116.3906的编码为1101 0010 1100 0100 0100。
        * 接下来将经度和纬度的编码合并，奇数位是纬度，偶数位是经度，得到编码 11100 11101 00100 01111 00000 01101 01011 00001。   

    ```java
    //https://github.com/wenhao/geohash
    package com.github.wenhao.geohash;

    import java.util.Arrays;
    import java.util.List;

    import com.github.wenhao.geohash.domain.Coordinate;

    public class GeoHash {

        private static final int MAX_PRECISION = 52;
        private static final long FIRST_BIT_FLAGGED = 0x8000000000000L;
        private long bits = 0;
        private byte significantBits = 0;
        private Coordinate coordinate;

        public GeoHash() {
        }

        public GeoHash(double latitude, double longitude) {
            this(latitude, longitude, MAX_PRECISION);
        }

        public GeoHash(double latitude, double longitude, int precision) {
            this.coordinate = new Coordinate(latitude, longitude);
            boolean isEvenBit = true;
            double[] latitudeRange = {-90, 90};
            double[] longitudeRange = {-180, 180};

            while (significantBits < precision) {
                if (isEvenBit) {
                    divideRangeEncode(longitude, longitudeRange);
                } else {
                    divideRangeEncode(latitude, latitudeRange);
                }
                isEvenBit = !isEvenBit;
            }
            bits <<= (MAX_PRECISION - precision);
        }


        public static GeoHash fromLong(long longValue) {
            return fromLong(longValue, MAX_PRECISION);
        }

        public static GeoHash fromLong(long longValue, int significantBits) {
            double[] latitudeRange = {-90.0, 90.0};
            double[] longitudeRange = {-180.0, 180.0};

            boolean isEvenBit = true;
            GeoHash geoHash = new GeoHash();

            String binaryString = Long.toBinaryString(longValue);
            while (binaryString.length() < MAX_PRECISION) {
                binaryString = "0" + binaryString;
            }
            for (int j = 0; j < significantBits; j++) {
                if (isEvenBit) {
                    divideRangeDecode(geoHash, longitudeRange, binaryString.charAt(j) != '0');
                } else {
                    divideRangeDecode(geoHash, latitudeRange, binaryString.charAt(j) != '0');
                }
                isEvenBit = !isEvenBit;
            }

            double latitude = (latitudeRange[0] + latitudeRange[1]) / 2;
            double longitude = (longitudeRange[0] + longitudeRange[1]) / 2;

            geoHash.coordinate = new Coordinate(latitude, longitude);
            geoHash.bits <<= (MAX_PRECISION - geoHash.significantBits);
            return geoHash;
        }

        public List<GeoHash> getAdjacent() {
            GeoHash northern = getNorthernNeighbour();
            GeoHash eastern = getEasternNeighbour();
            GeoHash southern = getSouthernNeighbour();
            GeoHash western = getWesternNeighbour();
            return Arrays.asList(northern, northern.getEasternNeighbour(), eastern, southern.getEasternNeighbour(),
                    southern, southern.getWesternNeighbour(), western, northern.getWesternNeighbour());
        }

        private GeoHash getNorthernNeighbour() {
            long[] latitudeBits = getRightAlignedLatitudeBits();
            long[] longitudeBits = getRightAlignedLongitudeBits();
            latitudeBits[0] += 1;
            latitudeBits[0] = maskLastNBits(latitudeBits[0], latitudeBits[1]);
            return recombineLatLonBitsToHash(latitudeBits, longitudeBits);
        }

        private GeoHash getSouthernNeighbour() {
            long[] latitudeBits = getRightAlignedLatitudeBits();
            long[] longitudeBits = getRightAlignedLongitudeBits();
            latitudeBits[0] -= 1;
            latitudeBits[0] = maskLastNBits(latitudeBits[0], latitudeBits[1]);
            return recombineLatLonBitsToHash(latitudeBits, longitudeBits);
        }

        private GeoHash getEasternNeighbour() {
            long[] latitudeBits = getRightAlignedLatitudeBits();
            long[] longitudeBits = getRightAlignedLongitudeBits();
            longitudeBits[0] += 1;
            longitudeBits[0] = maskLastNBits(longitudeBits[0], longitudeBits[1]);
            return recombineLatLonBitsToHash(latitudeBits, longitudeBits);
        }

        private GeoHash getWesternNeighbour() {
            long[] latitudeBits = getRightAlignedLatitudeBits();
            long[] longitudeBits = getRightAlignedLongitudeBits();
            longitudeBits[0] -= 1;
            longitudeBits[0] = maskLastNBits(longitudeBits[0], longitudeBits[1]);
            return recombineLatLonBitsToHash(latitudeBits, longitudeBits);
        }

        private GeoHash recombineLatLonBitsToHash(long[] latBits, long[] lonBits) {
            GeoHash geoHash = new GeoHash();
            boolean isEvenBit = false;
            latBits[0] <<= (MAX_PRECISION - latBits[1]);
            lonBits[0] <<= (MAX_PRECISION - lonBits[1]);
            double[] latitudeRange = {-90.0, 90.0};
            double[] longitudeRange = {-180.0, 180.0};

            for (int i = 0; i < latBits[1] + lonBits[1]; i++) {
                if (isEvenBit) {
                    divideRangeDecode(geoHash, latitudeRange, (latBits[0] & FIRST_BIT_FLAGGED) == FIRST_BIT_FLAGGED);
                    latBits[0] <<= 1;
                } else {
                    divideRangeDecode(geoHash, longitudeRange, (lonBits[0] & FIRST_BIT_FLAGGED) == FIRST_BIT_FLAGGED);
                    lonBits[0] <<= 1;
                }
                isEvenBit = !isEvenBit;
            }
            geoHash.bits <<= (MAX_PRECISION - geoHash.significantBits);
            geoHash.coordinate = getCenterCoordinate(latitudeRange, longitudeRange);
            return geoHash;
        }

        private long[] getRightAlignedLatitudeBits() {
            long copyOfBits = bits << 1;
            long value = extractEverySecondBit(copyOfBits, getNumberOfLatLonBits()[0]);
            return new long[]{value, getNumberOfLatLonBits()[0]};
        }

        private long[] getRightAlignedLongitudeBits() {
            long copyOfBits = bits;
            long value = extractEverySecondBit(copyOfBits, getNumberOfLatLonBits()[1]);
            return new long[]{value, getNumberOfLatLonBits()[1]};
        }

        private long extractEverySecondBit(long copyOfBits, int numberOfBits) {
            long value = 0;
            for (int i = 0; i < numberOfBits; i++) {
                if ((copyOfBits & FIRST_BIT_FLAGGED) == FIRST_BIT_FLAGGED) {
                    value |= 0x1;
                }
                value <<= 1;
                copyOfBits <<= 2;
            }
            value >>>= 1;
            return value;
        }

        private Coordinate getCenterCoordinate(double[] latitudeRange, double[] longitudeRange) {
            double minLon = Math.min(longitudeRange[0], longitudeRange[1]);
            double maxLon = Math.max(longitudeRange[0], longitudeRange[1]);
            double minLat = Math.min(latitudeRange[0], latitudeRange[1]);
            double maxLat = Math.max(latitudeRange[0], latitudeRange[1]);
            double centerLatitude = (minLat + maxLat) / 2;
            double centerLongitude = (minLon + maxLon) / 2;
            return new Coordinate(centerLatitude, centerLongitude);
        }

        private int[] getNumberOfLatLonBits() {
            if (significantBits % 2 == 0) {
                return new int[]{significantBits / 2, significantBits / 2};
            } else {
                return new int[]{significantBits / 2, significantBits / 2 + 1};
            }
        }

        private long maskLastNBits(long value, long n) {
            long mask = 0xffffffffffffffffl;
            mask >>>= (MAX_PRECISION - n);
            return value & mask;
        }

        private void divideRangeEncode(double value, double[] range) {
            double mid = (range[0] + range[1]) / 2;
            if (value >= mid) {
                addOnBitToEnd();
                range[0] = mid;
            } else {
                addOffBitToEnd();
                range[1] = mid;
            }
        }

        private static void divideRangeDecode(GeoHash geoHash, double[] range, boolean b) {
            double mid = (range[0] + range[1]) / 2;
            if (b) {
                geoHash.addOnBitToEnd();
                range[0] = mid;
            } else {
                geoHash.addOffBitToEnd();
                range[1] = mid;
            }
        }

        private void addOnBitToEnd() {
            significantBits++;
            bits <<= 1;
            bits = bits | 0x1;
        }

        private void addOffBitToEnd() {
            significantBits++;
            bits <<= 1;
        }

        public long toLong() {
            return bits;
        }

        public Coordinate coordinate() {
            return coordinate;
        }

        @Override
        public String toString() {
            if (significantBits % 5 == 0) {
                return String.format("bits: %s", Long.toBinaryString(bits));
            } else {
                return String.format("bits: %s", Long.toBinaryString(bits));
            }
        }
    }
    ``` 
    
    支持坐标转换成GeoHash与Long型值反转回坐标，转换会有误差，误差分米级能够接受。
    
2. 估算搜索范围起始值。

    * 12bit精度为：9775.7m， 11bit精度为：19551.5m ……
    * 算法, 如果用52位来表示一个坐标, 那么总共有: 2^26 * 2^26 = 2^52 个框:
        * 地球半径：radius = 6372797.560856m
        * 每个框的夹角：angle = 1 / 2^26 (2的26次方)
        * 每个框在地球表面的长度: length = 2 * π * radius * angle

3. 给出查询的中心坐标并计算其GeoHash值(52bit)。

4. 计算中心坐标相邻的8个坐标(中心坐标在两个框边界会有误差，此规避误差)。

5. 加上中心坐标共9个52bit的坐标值，针对每个坐标值参照搜索范围值算出区域值[MIN, MAX]。
    * 算法：MIN为坐标的搜索指定位起始长度后补零；MAX为坐标的搜索指定位终止长度后补壹。

6. 使用Redis命令ZRANGEBYSCORE key MIN MAX WITHSCORES查找。

7. 避免误差按照距离公式在将所有结果过滤一次(GeoHash反坐标再计算距离)。
