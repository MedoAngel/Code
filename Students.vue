<template>
    <v-container fluid>
        <v-row>
            <v-col cols="12" class="dialogContainer">
                <v-data-table show-select single-select :custom-filter="tableSearch" :search="searchText" @item-selected="rowSelected" @click:row="rowClicked" class="elevation-3" :headers="headers" :items="rows" item-key="uuid">
                    <template v-slot:top>
                        <v-container class="py-0" fluid>
                            <v-row>
                                <v-col class="py-0" cols="4">
                                    <v-card-title>{{$('Students')}}</v-card-title>
                                </v-col>
                                <v-col align="right" cols="8">
                                    <v-container fluid>
                                        <v-row justify="end">
                                            <v-col class="py-0" cols="8" md="6">
                                                <TableSearchBar v-model="searchText" @searchCol="updateSearchCol" :searchSelectCols="searchSelectCols" />
                                            </v-col>
                                            <v-col v-for="(btn,idx) in tableButtons" :key="idx" class="py-0" cols="1">
                                                <v-tooltip top>
                                                    <template #activator="{on}">
                                                        <v-btn v-on="on" small icon @click="btn.clickFn" class="elevation-3">
                                                            <v-icon :color="btn.color">
                                                               {{btn.icon}}
                                                            </v-icon>
                                                        </v-btn>              
                                                    </template>
                                                    {{btn.tooltip}}
                                                </v-tooltip>
                                            </v-col>
                                        </v-row>
                                    </v-container>
                                </v-col>
                            </v-row>
                        </v-container>
                    </template>
                </v-data-table>
                <v-dialog @click:outside="studentInfoFormDismissed" content-class="elevation-0 col-8 col-lg-5 pa-0" class="pa-0" v-model="showAddStudentForm">
                    <v-card scrollable>
                        <StudentInfoForm v-if="showAddStudentForm" :student="student" @studentAdded="studentAdded" @dismiss="studentInfoFormDismissed"></StudentInfoForm>
                    </v-card>
                </v-dialog> <!-- end of add student dialog-->
                <v-dialog @click:outside="studentPaymentTableDismissed" content-class="elevation-0 col-10 col-lg-6 pa-0" class="pa-0" v-model="showAddPaymentForm">
                    <v-card scrollable>
                        <StudentPaymentsTable v-if="showAddPaymentForm" :student="student"></StudentPaymentsTable>
                    </v-card>
                </v-dialog>
                <v-dialog content-class="elevation-0 col-6 col-lg-3 pa-0" class="pa-0" v-model="showRemoveConfirm">
                    <v-card scrollable>
                        <RemoveConfirm @result="isRemoveConfirmed" v-if="showRemoveConfirm"></RemoveConfirm>
                    </v-card>
                </v-dialog>
            </v-col>
        </v-row>
    </v-container>
</template>

<script>
import StudentInfoForm from '../components/StudentInfoForm.vue'
import StudentPaymentsTable from '../components/StudentPaymentsTable.vue'
import TableSearchBar from '../components/TableSearchBar.vue'
import RemoveConfirm from '../components/RemoveConfirm.vue'
import {mapState,mapActions} from 'vuex'
export default {
    components:{
        StudentInfoForm,
        StudentPaymentsTable,
        TableSearchBar,
        RemoveConfirm
    },
    computed:{
        ...mapState([
            '$DBS',
        ]),
        tableButtons:function(){
            return [
                {color:"yellow darken-2",icon:"mdi-account-cash",clickFn:this.addPaymentClicked,tooltip:this.$('Student Payments')},
                {color:"red lighten-1",icon:"mdi-account-minus mdi-flip-h",clickFn:this.removeStudentClicked,tooltip:this.$('Remove Student')},
                {color:"blue darken-1",icon:"mdi-account-edit",clickFn:this.editStudentClicked,tooltip:this.$('Edit Student Details')},
                {color:"green lighten-1",icon:"mdi-account-plus mdi-flip-h",clickFn:this.addStudentClicked,tooltip:this.$('Add Student')}
            ]
        },
        headers:function(){
            return [
                {text:"#",value:"number",sortable:false},
                {text:this.$("First Name"),value:"firstName"},
                {text:this.$("Last Name"),value:"lastName"},
                {text:this.$("Gender"),value:"gender"},
                {text:this.$("School Year"),value:"schoolYear"},
                {text:this.$("Groups"),value:"groups"},
                {text:this.$("Tuition Fees"),value:"payments"},
            ]
        },
        searchSelectCols:function(){
            // exclude certain items from the headers
            return this.headers.filter(obj => ["number","payments"].indexOf(obj.value) < 0)
        },
        rows:function(){
            return this.$DBS.students.array.map((student,idx) => ({
                uuid:student.uuid,
                number: idx+1,
                firstName: student.firstName||"",
                lastName: student.lastName||"",
                gender: student.gender.text||"",
                schoolYear: student.schoolYear.text||"",
                groups: student.groups.length ? student.groups.map(group => group.text).join(" , "):"",
                payments: this.itemPaymentsText(student.payments)
            }));
        }
    },
    data:function(){
        return {
            searchText:'',
            searchCol:'',
            isRowSelected:false,
            selectedRow:null,
            showAddStudentForm:false,
            showAddPaymentForm:false,
            showRemoveConfirm:false,
            defaultStudentInfo:{
                img:'',
                firstName:'',
                lastName:'',
                birthPlace:'',
                birthDate:'',
                gender:'',
                parents:[],
                payments:{},
                email:'',
                tel:'',
                mobile1:'',
                mobile2:'',
                schoolYear:'',
                groups:[],
                mentalHealthIssues:'',
                physicalHealthIssues:'',
                notes:''
            },
            student:this.defaultStudentInfo,
        }
    },
    methods:{
        ...mapActions([
            'ADD',
            'SNACKBAR',
            'REMOVE'
        ]),
        studentInfoFormDismissed:function(){
            this.showAddStudentForm = false;
        },
        studentPaymentTableDismissed:function(){
            this.showAddPaymentForm = false;
        },
        rowClicked:function(row,data){
            data.select(true);
        },
        rowSelected:function({item,value}){
            this.isRowSelected = value;
            this.selectedRow = this.$DBS.students.array.find(student => student.uuid == item.uuid);
            this.student = JSON.parse(JSON.stringify(value ? this.selectedRow : this.defaultStudentInfo));
        },
        studentAdded:function(student){
            if((student && this.selectedRow) && student.uuid == this.selectedRow.uuid){
                this.selectedRow = JSON.parse(JSON.stringify(student));
                this.student = JSON.parse(JSON.stringify(student));
            }
        },
        addPaymentClicked:function(){
            if(!this.isRowSelected){
                this.SNACKBAR({
                    show:true,
                    text:this.$('Please Select A Student From the Table')
                });
            }else{
                this.student = JSON.parse(JSON.stringify(this.selectedRow));
                this.showAddPaymentForm = true;
            }
        },
        addStudentClicked:function(){
            this.student=JSON.parse(JSON.stringify(this.defaultStudentInfo));
            this.showAddStudentForm=true;

        },
        editStudentClicked:function(){
            if(!this.isRowSelected){
                this.SNACKBAR({
                    show:true,
                    text:this.$('Please Select A Student From the Table')
                });
            }else{
                this.student = JSON.parse(JSON.stringify(this.selectedRow));
                this.showAddStudentForm = true;
            }
        },
        removeStudentClicked:function(){
            if(!this.isRowSelected){
                this.SNACKBAR({
                    show:true,
                    text:this.$('Please Select A Student From the Table')
                });    
            }else{
                this.showRemoveConfirm = true;
            }
        },
        removeStudent:function(){

            // find this student and get its groups
            this.$DBS.students.array.find(student => student.uuid == this.selectedRow.uuid).groups.map(group => {
                // get the group with this uuid 
                var thisGroup = this.$DBS.groups.array.find(Group => Group.uuid == group.value);
                // get the students of the group , filter them and reassign to group
                thisGroup.students = thisGroup.students
                // return an array excluding the student we just removed 
                .filter(studentUUID => studentUUID != this.selectedRow.uuid);

                this.ADD({
                    DB:this.$DBS.groups,
                    doc:thisGroup,
                    showSnackBar:false
                })
            });

            this.REMOVE({
                DB:this.$DBS.students,
                doc:this.selectedRow
            })
            this.selectedRow = null;
            this.isRowSelected = false;
            this.showRemoveConfirm = false;
        },
        isRemoveConfirmed:function(result){
            result ? this.removeStudent() : this.showRemoveConfirm = false;
        },
        itemPaymentsText:function(payments){
            var fees = this.calcTuitionFees(payments);
            return fees.due ? `${fees.paid}/${fees.due}`:"";
        },
        calcTuitionFees:function(payments){
            var totalTuitionFees = {due:0,paid:0};
            for(var groupUUID in payments){
                for(var month in payments[groupUUID].tuitionFees){
                    totalTuitionFees.due += Number(payments[groupUUID].tuitionFees[month].due);
                    totalTuitionFees.paid += payments[groupUUID].tuitionFees[month].paid.length ? payments[groupUUID].tuitionFees[month].paid.reduce((total,obj) => total+Number(obj.amount),0):0; 
                }
            }
            return totalTuitionFees;
        },
        updateSearchCol:function(col){
            this.searchCol = col;
        },
        tableSearch:function(value,search,item){
            return (search && item[this.searchCol].toString().toLowerCase().indexOf(search)>-1);
        }
    }
}
</script>

<style scoped>

</style>