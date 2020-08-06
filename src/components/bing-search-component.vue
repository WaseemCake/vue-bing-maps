<template>
    <div ref='searchBoxContainer' id="searchBoxContainer" class="customSearchBox">
        <div class="customSearchTitle">   {{this.options.searchTitle}}
            <input type= 'text' id= 'searchBox' ref='searchBox' class="searchBox"  autocomplete="off" placeholder="Search a port or a country and view cases associated with it"/>
        </div>
    </div>
</template>

<script>
    const searchEvents = Object.freeze([
        'click', 
    ]);
    import ComponentBase from '../mixins/component-base.vue';
    import Utils from '../utils.js';

    export default {
        name: 'bing-search',
        mixins: [ComponentBase],
        props: {
            options: {
                type: Object,
                required: false,
                default: () => {}
            },
            visible: {
                type: Boolean,
                required: false,
                default: () => true
            },
        },
        data(){
            return {
                itemType: 'search',
                searchTitle:this.options.searchTitle,
                allowedEvents: searchEvents
            };
        },
        inject: ['getMap'],
        methods:{
            render() {
                let map = this.getMap();
                let search = this.getItem();
                let self = this;
                let name = 'Microsoft.Maps.AutoSuggest'
                if(map) {
                return new Promise((resolve, reject) => {
                    if(search) {
                        self.destroy();
                    }

                    Promise.all([self.loadModule(name), self.$nextTick()]).then(() => {
                        let options = {
                            visible:self.visible,
                            maxResults: self.options.maxResults || 4,
                            map: map,
                        };
                        search = self.setItem(new Microsoft.Maps.AutosuggestManager(options));
                        search.attachAutosuggest(self.$refs.searchBox, self.$refs.searchBoxContainer, 
                            function (suggestionResult) {
                                map.entities.clear();
                                map.setView({ bounds: suggestionResult.bestView });
                        });
                        resolve();
                    }).catch(function(err){
                        reject(err);
                    }).finally(() => {
                        reject();
                    }); 
                });
                }
            },
            loadModule(name){
                let self = this;
                let normalizedName = (name || '').toLowerCase().trim();
                return new Promise((resolve, reject) => {
                    if(!normalizedName){
                        reject('A module name must be specified');
                    } else {
                        let loadedModules = Utils.loadedModules;
                        if(!loadedModules){
                            loadedModules = [];
                            Utils.loadedModules = loadedModules;
                        }
                        if(loadedModules.indexOf(normalizedName) > -1){
                            Utils.logger.log('AutoSuggest module ' + normalizedName + ' already loaded, resolving...');
                            resolve();
                        } else {
                            Microsoft.Maps.loadModule(name, {
                                callback(){
                                    Utils.logger.log('AutoSuggest module ' + normalizedName + ' successfully loaded!');
                                    loadedModules.push(normalizedName);
                                    resolve();
                                },
                                errorCallback(message){
                                    Utils.logger.warn('AutoSuggest module ' + normalizedName + ' loading failed!');
                                    reject(message);
                                }
                            })
                        }
                    }
                });                
            },            
            destroy(){
                let search = this.getItem();

                try {
                    if(search) {
                        let map = this.getMap();

                        if(map){
                            //map.remove(search);
                        }
                    }
                }finally{
                    this.setItem(null);
                }
            }
        },
        mounted(){
            Utils.logger.log('mounted lifecycle hook, rendering search' + this._uid + '...');
            this.render();
        },
        beforeUpdate(){
            Utils.logger.log('beforeUpdate lifecycle hook, search ' + this._uid);
        },
        updated(){
            Utils.logger.log('updated lifecycle hook, search '+ this._uid);
        },
        beforeDestroy(){
            Utils.logger.log('beforeDestroy lifecycle hook, destroying search ' +  this._uid + '...');
            this.destroy();
        }
    }
</script>