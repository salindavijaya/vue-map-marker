<script>
/* eslint-disable no-undef */
import { computed , ref , onMounted , onUnmounted ,onBeforeUpdate} from 'vue'
import { useGeolocation } from './useGeolocation'
import { Loader } from '@googlemaps/js-api-loader'
import hubList from '../assets/hubs.json'

const API_KEY = 'AIzaSyBWxZq6maj1gxS3U6AglR7rvw5yQgPseYc'

export default {
    name: 'MapComponent',
    setup() {
        const { coords } = useGeolocation()  //user-defined dunction for get geolocation
        const currPos = computed( () => ({lat:coords.value.latitude,lng:coords.value.longitude}))
        const loader = new Loader({ apiKey: API_KEY })
        const otherPos = ref(null)
        const mapDiv = ref(null)
        let map = ref(null)
        let selectedM = ref('')
        let clickListener = null
        let items = ref([])
        let markList = ref([])
        let defaultMarkList = ref([])
        onMounted(async () => {
            await loader.load()
            //initializing map
            map.value = new google.maps.Map(mapDiv.value, {
                center: currPos.value,
                zoom: 10,
                mapTypeId: google.maps.MapTypeId.ROADMAP,
                disableDefaultUI: true
                
            })

            //adding map markers using json file
            hubList.data.map(async (hub,index) => {
                await    new Promise((resolve)=>{
                    //request lat,lng by address using geocode service
                        var geocoder = new google.maps.Geocoder();
                        geocoder.geocode( { 'address': hub.name},function(results, status) {
                            if (status == 'OK') {
                                //adding marker
                                let marker = new google.maps.Marker({
                                    map: map.value,
                                    name: hub.name,
                                    position: results[0].geometry.location,
                                    label:String.fromCharCode(97 + index),
                                    icon: {
                                        path: google.maps.SymbolPath.CIRCLE,
                                        scale: 9.5,
                                        fillColor: "#f2920c",
                                        fillOpacity: 0.8,
                                        strokeWeight: 0.4
                                    }
                                })
                                //adding marker click listener
                                marker.addListener("click", () => {
                                    map.value.setZoom(12);
                                    map.value.setCenter(marker.getPosition());
                                    selectedM.value = marker.name
                                    markList.value = []
                                    if(!items.value.find(a=>a.name == marker.name)){
                                        items.value.push({name:marker.name,label:marker.label,icon:marker.icon})
                                        defaultMarkList.value.push(marker)
                                        marker.icon = null
                                        marker.label = ''
                                    }else{
                                        let markPre = items.value.find(a=>a.name == marker.name)
                                        if(markPre){
                                            items.value = items.value.filter(f=>f.name != marker.name)
                                            marker.icon = markPre.icon
                                            marker.label = markPre.label
                                        }
                                    }
                                    markList.value = items.value
                                    new google.maps.Marker(marker)
                                })
                                //exporting values to calculate distance
                                resolve(
                                    {
                                        location:results[0].geometry.location,
                                        label:String.fromCharCode(97 + index)
                                    }
                                )
                            }
                        });
                    }).then(async e => {
                    await    new Promise((resolve)=>{
                        //request distance 
                        var distance = new google.maps.DistanceMatrixService();
                            distance.getDistanceMatrix(
                                {
                                    origins: [currPos.value],
                                    destinations: [e.location],
                                    travelMode: 'DRIVING'

                                }, d => {
                                    //exporting values to list-out 
                                    resolve({
                                        distance:d.rows[0].elements[0].distance.value,
                                        name:hub.name,
                                        label:e.label,
                                        marker:hub})
                            }); 
                        }).then(data=>{
                            //pushing list elements
                            markList.value.push(data)
                        })
                    })
                    
            })
            //mark my location on map
             new google.maps.Marker({
                position: currPos.value,
                map:map.value,
                title: "My Location",
                icon: {
                    path: google.maps.SymbolPath.CIRCLE,
                    scale: 9.5,
                    fillColor: "#064fc4",
                    fillOpacity: 0.8,
                    strokeWeight: 0.4
                }
            }) 
        })
        onUnmounted(async () => {
            if (clickListener) clickListener.remove()
        })
        onBeforeUpdate(async ()=>{
            // sorting Hubs by distance
            markList.value.sort((a,b)=>a.distance - b.distance)
            console.log('after sort',markList.value)
        })

        return {
            currPos, otherPos, mapDiv ,markList ,selectedM
        }
    },
    methods:{
        onClickSearch:()=>{window.location.reload()}
    }
    
}
</script>
<template>
<div class="row d-flex justify-content-center">
    <div class="col-sm-12 col-md-8 col-lg-6 col-xl-6">
        <div class="card">
            <div class="card-hearder mt-4">
                <h4><strong>Lacate Hub</strong></h4>
            </div>
            <div class="card-body">
                <div>
                   <div ref="mapDiv" style="width: 100%; height: 30vh" />
                </div>
            </div>
        </div>
        <div class="card">
            <div class="container text-start mt-2">
                <h5> <strong> Hubs Near You</strong></h5>
            </div>
            <div class="card-body overflow-auto" style="max-height:30vh">
                <ul class="list-group" >
                <li class="list-group-item"  v-for="(item, index) in markList" :key="index" :ref="markList">
                    <span class="float-start">{{item.name}}</span>
                    <span class="float-end rounded-circle text-light  opacity-75" style="width:25px;background-color:#f2920c">{{item.label}}</span>
                </li>
                </ul>
            </div>
            <div class="card-footer">
                <button  class="btn btn-secondary" @click="onClickSearch">Search Hubs</button>
            </div>
        </div>
    </div>
</div>
</template>

