<template>
    <div class="SearchCity">
        <!-- 热门城市 -->
        <div v-if="showHotList" class="hotCity">
            <Row :gutter="12" class="curLocation">
                <Col :span="5" style="padding-left:0">
                <label>当前定位</label>
                </Col>
                <Col :span="15">
                    <span :class="{c_b2b2b2: location.city === '未定位'}">{{location.city || "未定位"}}</span>
                </Col>
                <Col :span="4" style="display: flex; text-align:center; align-items: center; justify-content: center;">
                <LoadingBlack v-if="locationing" height="21" width="21" color="#999" />
                <Location2 v-else height="21" width="21" color="#999" @click.native="getLocation()" />
                </Col>
            </Row>
            <p class="labelForHot">热门城市</p>
            <Row :gutter="12" class="hotCityList">
                <ul>
                    <li v-for="(item, index) in hotCityList" :key="index" @click="selectCity(item)">{{ item }}</li>
                </ul>
            </Row>
        </div>
        <!-- /热门城市 -->
        <template v-else>
            <!-- 搜索列表 -->
            <div v-if="formatList.length > 0" class="cityList">
                <ul>
                    <li v-for="(name, index) in formatList" :key="index" @click="selectCity(name)">{{ name }}</li>
                </ul>
            </div>
            <!-- /搜索列表 -->
            <div v-else :class="$style.nothing">
                <img :src="nothingImg" alt="空空如也">
            </div>
        </template>
    </div>
</template>
<script>
import Location2 from '../../icons/Location2';
import LoadingBlack from '../../icons/LoadingBlack';

import getCurLocation from '../../utils/getLocation';
import { NOTICE } from '../../stores/types';

import request, { createAPI, addAccessToken } from '../../utils/request';
import { resolveImage } from '../../utils/resource';
const nothingImg = resolveImage(require('../../statics/images/defaultNothingx3.png'))


export default {
    name: 'SearchCity',
    props: {
        keyword: String,
        datas: Array
    },
    components: {
        Location2,
        LoadingBlack
    },
    data() {
        return({
            locationing: true,
            location: {
                city: '',
                lat: null,
                lng: null
            },
            hotCityList: [],
            nothingImg
        })
    },
    computed: {
        showHotList() {
            return !(this.keyword.length > 0 || this.datas.length > 0);
        },
        formatList() {
            return this.datas.map((item, index) => {
                let name = '';
                item = item.tree;
                while(item) {
                    name = item.name + "," + name;
                    item = item.parent;
                }
                return name.substr(0, (name.length - 1));
            })
        }
    },
    methods: {

        getLocation() {
            this.locationing = true;
            this.$storeLocal.remove('LocationObj');
            const that = this;
            getCurLocation({
                success: that.locationSuccess,
                error: that.locationError
            })
        },
        locationSuccess(data) {
            this.locationing = false;

            const { addressComponent: { city = '', street = '', streetNumber = '' } = {}, position: { lat = '', lng = '' } = {} } = data

            this.location = {
                lat,
                lng,
                city: street ? (street + streetNumber) : city
            }

            this.$storeLocal.set('LocationObj', this.location)
        },
        locationError(error) {
            this.locationing = false
            this.$store.dispatch(NOTICE, cb => {
                cb({
                    show: true,
                    time: 2000,
                    status: false,
                    text: error
                })
            })
        },

        selectCity(city) {
            if(city !== this.city) {
                request.get(createAPI(`around-amap/geo?address=${city.replace(/[\s\uFEFF\xA0]+/g, '')}`)).then(res => {
                    const { data: { geocodes: [{ location = '0,0' } = {}] = [] } = {} } = res
                    const [lng, lat] = location.split(',')
                    this.location = {
                        city: city.split(",").reverse()[0],
                        lng,
                        lat
                    }

                    this.$storeLocal.set('LocationObj', {
                        city: city.replace(/[\s\uFEFF\xA0]+/g, ',').split(",").reverse()[0],
                        lng,
                        lat
                    });

                    this.$emit('input', this.location.city);
                    this.$emit('closeSearch');
                    this.$bus.emit('UpdateLocation');
                }).catch(err => {
                    console.log(err);
                });
            }
        }
    },
    created() {

        const { lat, lng, city } = this.$storeLocal.get('LocationObj') || {}

        if(!isNaN(lat + lng) && typeof city === 'string') {
            this.location = {
                lat,
                lng,
                city
            }
            this.locationing = false
        } else {
            // 延迟 .5s 定位
            setTimeout(() => {
                this.getLocation()
            }, 500)
        }

        if(this.showHotList) {
            request.get(createAPI(`locations/hots`)).then(({ data = [] }) => {
                this.hotCityList = [
                    ...data
                ]
            })
        }
    }
}
</script>
<style lang="less" module>
.nothing {
    width: 100%;
    display: flex;
    font-size: 24px;
    align-items: center;
    justify-content: center;
    text-align: center;
    color: #ccc; // background-color: #fff;
    >img {
        margin: 30%;
        width: 70%;
    }
}
</style>
<style lang="scss">
.t_c {
    text-align: center;
}

.findCityList {
    background-color: #fff;
}

.c_b2b2b2 {
    color: #b2b2b2;
}

.hotCity {
    height: 100%;
    background-color: #f4f5f5;
    .curLocation {
        margin-top: 20px;
        padding: 0 10px;
        height: 45px;
        line-height: 45px;
        background-color: #fff;
        label {
            color: #333;
        }
    }
    .hotCityList {
        padding: 10px;
        width: 100%;
        font-size: 0;
        background-color: #fff;
        ul {
            margin-left: -15px;
            margin-top: 15px;
            li {
                overflow: hidden;
                display: inline-block;
                margin-left: 15px;
                margin-bottom: 15px;
                padding: 5px;
                border-radius: 4px;
                height: 30px;
                font-size: 14px;
                text-align: center;
                text-overflow: ellipsis;
                background-color: #f4f5f5;
                white-space: nowrap;
            }
        }
    }
    .labelForHot {
        padding: 8px 10px;
        font-size: 14px;
    }
}

.cityList {
    background-color: #fff;
    ul {
        margin: 0;
        padding: 0 10px;
        li {
            width: 100%;
            border-bottom: 1px solid #ededed;
            height: 50px;
            line-height: 49px;
        }
    }
}
</style>