/* eslint-disable */
<template>
    <v-container>
        <v-row class="text-center">
            <v-col cols="2">
                <v-img :src="require('../assets/logo_baseTVN.png')" class="my-3" contain height="100"/>
            </v-col>
            <v-col class="mb-4">
                <h2 class="display-1 font-weight-bold mb-2 blue--text text--lighten-1">Declaracion de Ingresos Brutos</h2>
            </v-col>
        </v-row>       
            <v-form>
                <v-row>
                <v-subheader>Informacion del Contribuyente</v-subheader>
                </v-row>
                <v-row>
                <v-col
                cols="12"
                sm="12"
                md="12">
                <v-text-field 
                    v-model="this.razonSocial"
                    label="Razon Social"
                    readonly
                 
                >
                </v-text-field>                         
                </v-col>
                <v-col
                cols="12"
                sm="6"
                md="9"
                >
                <v-text-field 
                    v-model="this.domicilio"
                    label="Dirección"
                    readonly
                 
                >
                </v-text-field>                         
                </v-col>
                <v-col
                cols="12"
                sm="6"
                md="3"
                >
                <v-text-field 
                    v-model="this.telefono"
                    label="Contacto"
                    readonly
                 
                >
                </v-text-field>                                         
                </v-col>
                </v-row>
            </v-form>
            <v-card class="mx-auto mt-5" max-width="1200">        
                <v-data-table
                    :headers="columnas"
                    :items="movprodsalientes"
                    class="elevation-19"
                    :items-per-page="12"
                    :footer-props="{
                        'items-per-page-options': [12]
                    }" 
                    
                >   // ACA SE DECLARAN LAS FUNCIONES DE FORMATEO DE FECHAS
                    <template v-slot:[`item.FECHA_VENC`]="{ item }">
                        <span>{{ formatearFecha(item.FECHA_VENC)}}</span>
                    </template>
                    <template v-slot:[`item.FECHA_DECLARACION`]="{ item }">
                        <span>{{ formatearFecha(item.FECHA_DECLARACION)}}</span>
                    </template>
                    <template v-slot:[`item.FECHA_PAGO`]="{ item }">
                        <span>{{ formatearFechaConHora(item.FECHA_PAGO)}}</span>
                    </template>
                    
                    // ACA SE DEFINE EL TEXTFIELD PARA EDICION DE BASE IMPONIBLE
                    <template v-slot:[`item.BASE_IMPONIBLE`]="{ item }">
                        <v-text-field @keypress="isNumber($event, editedItem.BASE_IMPONIBLE)"  v-model="editedItem.BASE_IMPONIBLE" :hide-details="true" dense single-line :autofocus="true" v-if="item.RECNO === editedItem.RECNO"></v-text-field>
                        <span v-else>{{item.BASE_IMPONIBLE}}</span>
                    </template>   
                    // ACA SE DEFINEN LOS BOTONES DE ACCION DE LA TABLA                 
                    <template v-slot:[`item.actions`]="{ item }">
                        <div v-if="item.RECNO === editedItem.RECNO">
                            <div @click="close(item)">
                                <v-tooltip top>
                                    <template v-slot:activator="{ on, attrs }">
                                            <v-icon color="red" class="mr-3"    v-bind="attrs" v-on="on">
                                                mdi-window-close
                                            </v-icon>
                                    </template>
                                    <span>Cancelar</span>
                                </v-tooltip>
                            </div> 
                            <div v-if="item.BASE_IMPONIBLE <= 0" @click="calcularIIBB(editedItem.BASE_IMPONIBLE,item)">
                                <v-tooltip top>
                                    <template v-slot:activator="{ on, attrs }">
                                            <v-icon color="green"   v-bind="attrs" v-on="on">
                                                mdi-check
                                            </v-icon>
                                    </template>
                                    <span>Pre-calcular impuesto</span>
                                </v-tooltip>
                            </div> 
                            <div v-if="item.IMPORTE_IIBB > 0" @click="save">
                                <v-tooltip top>
                                    <template v-slot:activator="{ on, attrs }">
                                            <v-icon color="green"   v-bind="attrs" v-on="on">
                                                mdi-file-document-outline
                                            </v-icon>
                                    </template>
                                    <span>Facturar impuesto calculado</span>
                                </v-tooltip>
                            </div>  
                        </div>
                        <div v-else-if="item.Seleccionado == 'primero declarar' && validacionEdicionHabil(item.FECHA_VENC)" color="green" class="mr-3" @click="editItem(item)" >
                            <v-tooltip top>
                                <template v-slot:activator="{ on, attrs }">
                                        <v-icon color="green"   v-bind="attrs" v-on="on">
                                            mdi-pencil
                                        </v-icon>
                                </template>
                                <span>Ingresar base imponible</span>
                            </v-tooltip>
                        </div> 
                        <div v-if="item.Seleccionado == 'declarado'" color="green" class="mr-3" >
                            <v-tooltip top>
                                <template v-slot:activator="{ on, attrs }">
                                        <v-icon color="green"   v-bind="attrs" v-on="on">
                                            mdi-check-all
                                        </v-icon>
                                </template>
                                <span>Ingreso bruto declarado</span>
                            </v-tooltip>
                        </div>
                    </template>
                </v-data-table>
            </v-card>
            <!-- componente snackbar para mostrar mensaje de DDJJ Exitosa -->
            <v-snackbar v-model="snackbar" color="green accent-4"> {{textsnack}}
                <template v-slot:action="{attrs}">
                    <v-btn text v-bind="attrs" @click="snackbar = false">Cerrar</v-btn>
                </template>
            </v-snackbar>  
        </v-container>        
</template>
<script>
import axios from 'axios';
import moment from 'moment';
export default {
    name: 'listarIIBB-item',
    contribuyente: {
        razonSocial: String,
        domicilio: String,
        telefono: String
    },
methods: {    
    ////validacion de entrada numerica
    isNumber: function(evt, item)  {
        const keysAllowed = ['0', '1', '2', '3', '4', '5', '6', '7', '8', '9', '.'];
        const keyPressed=  evt.key;
        
        if (!keysAllowed.includes(keyPressed) || this.validarDecimales(keyPressed, item)) {
            evt.preventDefault()
        }
    },
    validarDecimales(key, item) {
        if(item.includes('.')){
            if(item.length-item.indexOf('.') > 2){
                console.log("more")
                return true
            }
        }
        return false
    },
// METODO DE CARGA DE LA GRILLA
    async obtenerIIBBContribuyente()
    {
        this.id = this.$route.params.id;  
        var url = 'http://localhost:8090/api/ingresosbrutos/'+ this.id;
        var ultimaFecha = null;
        axios.get(url).then((respuesta)=>{
            this.movprodsalientes  = respuesta.data;   
            respuesta.data.forEach(function(item) {
                
                if(item.IMPORTE_IIBB == null && ultimaFecha == null){
                    ultimaFecha = item.FECHA_VENC;
                    this.lastDate = item.FECHA_VENC;
                }
                //fomateo del ingreso bruto
                if(item.IMPORTE_IIBB.toString().includes('.')){
                    var valueFloat = item.IMPORTE_IIBB.toString()+'00' //se agregarn ceros para evitar numeros con decimales de menos
                    var splitIndex = valueFloat.indexOf('.') + 3
                    var mediumValue = parseFloat(valueFloat.substr(0,splitIndex) + '5')
                    
                    if(mediumValue<item.IMPORTE_IIBB){
                        item.IMPORTE_IIBB = (item.IMPORTE_IIBB + 0.01).toString().substr(0,splitIndex)
                    }else{
                        item.IMPORTE_IIBB = valueFloat.substr(0,splitIndex)
                    }
                }else{
                    item.IMPORTE_IIBB = item.IMPORTE_IIBB.toString()+'.00'
                }
                //formateo de la base imponible
                if(item.BASE_IMPONIBLE.toString().includes('.')){
                    var valueFloatB = item.BASE_IMPONIBLE.toString()+'00' //se agregarn ceros para evitar numeros con decimales de menos
                    var splitIndexB = valueFloatB.indexOf('.') + 3
                    var mediumValueB = parseFloat(valueFloatB.substr(0,splitIndexB) + '5')
                    
                    if(mediumValueB<item.BASE_IMPONIBLE){
                        item.BASE_IMPONIBLE = (item.BASE_IMPONIBLE + 0.01).toString().substr(0,splitIndexB)
                    }else{
                        item.BASE_IMPONIBLE = valueFloatB.substr(0,splitIndexB)
                    }
                }else{
                    item.BASE_IMPONIBLE = item.BASE_IMPONIBLE.toString()+'.00'
                }
            }); 
        });
        return ultimaFecha;

    },
    // METODOS DE FORMATEO DE FECHA
    formatearFecha(fecha) {
        if (moment(fecha).isValid())
        {
            fecha = moment(fecha).add(1,"d")
            return moment(fecha).format("DD/MM/YYYY")
        }
    },
    formatearFechaConHora(fecha) {
            if (moment(fecha).isValid())
            {
                return moment(fecha).format("DD/MM/YYYY HH:mm")
                
            }
    },
    mesActual(){
        return moment().format("MM")
    },
    validacionEdicionHabil(itemFecha){
        var mesItem = moment(itemFecha).format("MM")
        if(mesItem<this.mesActual()+1){
            return true
        }
        return false
    },
    // METODO DE REDONDEO DE DECIMALES PARA GRILLA
    redondeo2decimales(num, decimales = 2) {
        var signo = (num >= 0 ? 1 : -1);
        num = num * signo;
        if (decimales === 0) //con 0 decimales
            return signo * Math.round(num);
        // round(x * 10 ^ decimales)
        num = num.toString().split('e');
        num = Math.round(+(num[0] + 'e' + (num[1] ? (+num[1] + decimales) : decimales)));
        // x * 10 ^ (-decimales)
        num = num.toString().split('e');
        return signo * (num[0] + 'e' + (num[1] ? (+num[1] - decimales) : -decimales));
    },
     // METODOS DE EDICION DE BASE IMPONIBLE EN LA TABLA
     editItem (item) {
         item.BASE_IMPONIBLE = ''   
      this.editedIndex = this.movprodsalientes.indexOf(item);
      this.editedItem = Object.assign({}, item);   
      
    },
    
    close (item) {
      setTimeout(() => {
        this.editedItem = Object.assign({}, this.defaultItem);
        this.editedIndex = -1;
        item.BASE_IMPONIBLE = 0;
        item.ALICUOTA = '';
        item.IMPORTE_IIBB = '';
      }, 300)
    },
    save () {
      if (this.editedIndex > -1) {
        try{
            var url = 'http://localhost:8090/api/altaFactura?baseImponible='
            + this.editedItem.BASE_IMPONIBLE
            +'&idContribuyente='
            +this.id
            +'&recnoMontoProductoSaliente='
            +this.editedItem.RECNO;
            console.log(url);
            let respuesta = axios.put(url).then(this.snackbar = true)
            if (respuesta.status === 201)
            {
                console.log(respuesta.status);
            }
            setTimeout(() => {
                console.log("recargando")
                //this.movprodsalientes = []// se vacia la tabla para actualizarla
                this.obtenerIIBBContribuyente();
            }, 1000)
        }catch(error){
            console.log(error);
        }
        //Object.assign(this.movprodsalientes[this.editedIndex], this.editedItem)
      }
      this.close()
    },
    obtenerAlicuota() {
        var url = 'http://localhost:8090/api/alicuota/';
        
        axios.get(url).then(response =>{            
            this.alicuota = response.data.alicuota;                      
        });        
    },
    async obtenerImporteIIBB(baseImp) {
        try{
            var url = 'http://localhost:8090/api/calcularMontoIIBB?alicuota='+this.alicuota+'&monto='+baseImp
            let respuesta = await axios.get(url);
            this.importeIIBB  = respuesta.data.monto; 
            this.editedItem.IMPORTE_IIBB =  this.redondeo2decimales(this.importeIIBB);
            Object.assign(this.movprodsalientes[this.editedIndex], this.editedItem);
        } catch(error) {
            console.log(error);
        }
    },
    calcularIIBB(baseImp,item)
    {
        if (baseImp > 0 )
        {
            this.obtenerImporteIIBB(baseImp);
            this.editedIndex = this.movprodsalientes.indexOf(item);              
            this.editedItem.ALICUOTA = this.alicuota;
        }
    },
    async buscarDatosContribuyente()
    {
        try{
            var url = 'http://localhost:8090/api/contribuyentes/'+this.$route.params.cuit;
            console.log(url)
            await axios.get(url).then(response =>{
                this.razonSocial = response.data[0].UMENM
                this.domicilio = response.data[0].UMEDI
                this.telefono = response.data[0].UMETE
            })
        } catch(error) {
            console.log(error);
        }
    }
},
async created(){       
        this.obtenerIIBBContribuyente();
        this.obtenerAlicuota();
        this.buscarDatosContribuyente();
},
data(){        
          return{ 
            columnas: [
                    { text: 'RECNO', value: 'RECNO', align: ' d-none', class: 'light-blue darken-4 white--text', sortable: false},
                    { text: 'PERIODO', value: 'PERIODO', class: 'light-blue darken-4 white--text', sortable: false},
                    { text: 'FECHA VENCIMIENTO', value: 'FECHA_VENC',  class: 'light-blue darken-4 white--text', sortable: false},
                    { text: 'BASE IMPONIBLE', value: 'BASE_IMPONIBLE', class: 'light-blue darken-4 white--text',align: 'right', sortable: false},
                    { text: 'ALICUOTA', value: 'ALICUOTA', class: 'light-blue darken-4 white--text',align: 'right', sortable: false},
                    { text: 'IMPORTE IIBB', value: 'IMPORTE_IIBB', class: 'light-blue darken-4 white--text',align: 'right', sortable: false},
                    { text: 'FECHA DECLARACION', value: 'FECHA_DECLARACION', class: 'light-blue darken-4 white--text', sortable: false},
                    { text: 'ID FACTURA', value: 'ID_FACTURA', class: 'light-blue darken-4 white--text',align: 'right', sortable: false},
                    { text: 'FECHA PAGO', value: 'FECHA_PAGO', class: 'light-blue darken-4 white--text', sortable: false},
                    { text: 'ACCIONES', value: 'actions', class: 'light-blue darken-4 white--text', sortable: false , width: "100px"},
                    { text: 'Seleccionado', value: 'Seleccionado', align: ' d-none', class: 'light-blue darken-4 white--text', sortable: false},
               ],  
               razonSocial:'',
               domicilio:'',
               telefono:'',
               snackbar:false,
               textsnack:'¡La declaracion de Ingresos Brutos ha sido exitosa!',      
               alicuota: '',
               importeIIBB:'',             
               movprodsalientes: [],
               editedIndex: -1,
               ultimaFecha:null,
                editedItem: {
                    RECNO: 0,
                    PERIODO: '',
                    FECHA_VENC: '',
                    BASE_IMPONIBLE:0,
                    ALICUOTA: 0,
                    IMPORTE_IIBB: 0,
                    FECHA_DECLARACION:'',
                    ID_FACTURA: 0,
                    FECHA_PAGO: '',
                    Seleccionado:false
                },
                defaultItem: {
                    RECNO: 0,
                    PERIODO: '',
                    FECHA_VENC: '',
                    BASE_IMPONIBLE:0,
                    ALICUOTA: 0,
                    IMPORTE_IIBB: 0,
                    FECHA_DECLARACION:'',
                    ID_FACTURA: 0,
                    FECHA_PAGO: '',
                    Seleccionado:false
              },   
          
          }
          
    }
}
</script>