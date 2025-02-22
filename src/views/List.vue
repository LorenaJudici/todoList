<template>
  <div class="container mt-2">

      <b-form inline class="mb-2">
          <b-form-input 
            v-model="filter.subject" 
            id="subject" 
            placeholder="ex: lavar carro"
            class="mr-2"
            autocomplete="off"
            >
        </b-form-input>

        <b-form-select
            v-model="filter.status"
            :options="optionsList"
            class="mr-2"
        ></b-form-select>

        <b-button 
            variante="outline-secondary"
            @click="filterTasks"
            class="mr-2"
            >
            <i class="fa-solid fa-search"></i>
        </b-button>

            <b-button 
            variante="outline-secondary"
            @click="clearFilter"
            class="mr-2"
            >
            <i class="fa-solid fa-times"></i>
        </b-button>
      </b-form>

      <template v-if="isLoading">
          <div class="loading-spin">
              <b-spinner style="width: 5rem; height: 5rem;">
              </b-spinner>

          </div>
      </template>
      <template v-if="isTasksEmpty" && !isLoading> 
        <div class="empty-data mt-2">
            <img src="../assets/images/empty-data.svg" class="empty-data-image">
            <b-button variant="outline-primary" class="mt-2" to="/form">Criar tarefa</b-button>
        </div>
      </template>

      <template v-if="!isTasksEmpty" && !isLoading>
          <!-- Quando quiser usar localbase substitui abaixo por <div v-for="(task, index) in tasks" :key="index"> -->
        <div v-for="(task) in tasks" :key="task.id">
                <b-card 
                    class="mb-2"
                    :class="{'finished-task' : isFinished(task)}">

                <div class="d-flex justify-content-between">
                     <b-card-title>{{task.subject}}</b-card-title>

                     <span>
                         <b-badge
                            :variant="variantOverdue(task.dateOverdue, task.status)"
                         >{{overduePresenter(task.dateOverdue)}}</b-badge>
                     </span>
                </div>

                <b-card-text>
                    {{task.description}}
                </b-card-text>

                 <b-button 
                    variant="outline-secondary" 
                    class="mr-2" 
                    @click="updateStatus(task.id, status.FINISHED )"
                    v-b-tooltip.hover
                    title="Concluir"
                    
                    > <!--Quando quiser usar localbase @click="edit(index)"-->
                    <i class="fa-solid fa-check"></i>
                </b-button>
                
                 <b-button 
                    variant="outline-secondary" 
                    class="mr-2" 
                    @click="updateStatus(task.id, status.ARCHIVED )"
                    v-b-tooltip.hover
                    title="Arquivar"
                    > <!--Quando quiser usar localbase @click="edit(index)"-->
                    <i class="fa-solid fa-archive"></i>
                </b-button>

                <b-button 
                    variant="outline-secondary"
                    class="mr-2" 
                    @click="edit(task.id)"
                    v-b-tooltip.hover
                    title="Editar"
                    > <!--Quando quiser usar localbase @click="edit(index)"-->
                    <i class="fa-solid fa-edit"></i>
                </b-button>

                <b-button 
                    variant="outline-danger" 
                    class="mr-2" 
                    @click="remove(task.id)"
                    v-b-tooltip.hover
                    title="Excluir"
                    ><!--Quando quiser usar localbase @click="edit(index)"-->
                    <i class="fa-solid fa-times"></i>
                </b-button>
            </b-card>
        </div>

      </template>
      

  <b-modal ref="modalRemove" hide-footer title="Exclusão de tarefa">
      <div class="d-block text-center">
          Deseja realmente excluir essa tarefa? {{taskSelected.subject}}

      </div>
      <div class="mt-3 d-flex justify-content-end">
            <b-button variant="outline-secondary" class="mr-2" @click="hideModal">
            Cancelar
          </b-button>

          <b-button variant="outline-danger" class="mr-2" @click="confirmRemoveTask">
            Excluir
          </b-button>
      </div>

  </b-modal>
  </div>
</template>

<script>
import TasksModel from '@/models/TasksModel';
import Status from "@/valueObjects/status";
import ToastMixin from "@/mixins/toastMixin.js";

export default {
    name:"List",
    mixins: [ToastMixin],

    data()  {
        return{
               tasks: [],
               taskSelected:[],
               status:Status,
               filter:{
                   subject:null,
                   status:null,
                   
               },
               optionsList:[
                        {value:null,text:"Selecione algum status"},
                        {value:Status.OPEN,text:"Aberto"},
                        {value:Status.FINISHED,text:"Concluido"},
                        {value:Status.ARCHIVED,text:"Arquivado"}
                    ],
                isLoading: false,
                statusList:[Status.OPEN, Status.FINISHED]
            };
    },

    async created(){ //Colocando o async na frente para usa a API
        //Pegando os dados da API Fake
         this.tasks = await this.getTasks();
        //Usando localbase
        //this.tasks = (localStorage.getItem("tasks"))? JSON.parse(localStorage.getItem("tasks")) : [];
    },

    methods:{
        //Quando quiser usar localbase trocar abaixo por edit(index)
        edit(taskId){
            this.$router.push({name:"form", params:{taskId}}) // para usar localbase colocar params:{index}
        },
        //Quando usar com localbase abaixo troca por remove(task,index)
         async remove(taskId){
            this.taskSelected = await TasksModel.find(taskId); // Quando localbase this.taskSelected = task;
            //this.taskSelected.index = index; Quando localbase
            this.$refs.modalRemove.show();
        },
        hideModal(){
            this.$refs.modalRemove.hide();
        },
        async confirmRemoveTask(){
             this.taskSelected.delete();
             this.tasks = this.tasks = await this.getTasks();
            // this.tasks.splice(this.taskSelected.index, 1); Quando localbase 
            //localStorage.setItem("tasks", JSON.stringify(this.tasks)); Quando localbase
            this.hideModal();
        },

        async updateStatus(taskId, status){
            let task = await TasksModel.find(taskId);
            task.status = status;
            await task.save();

            this.tasks = this.tasks = await this.getTasks();
            this.showToast("sucess" , "Sucesso", "Staus da tarefa atualizado com sucesso");
        },

        isFinished(task){
            return task.status === this.status.FINISHED;
        },

        async filterTasks(){
            let filter = { ... this.filter}; // pega tudo o que tem no filtro
            filter = this.clean(filter);
            filter.userId = JSON.parse(localStorage.getItem('authUser')).id,
             this.tasks = await TasksModel.params(filter).get(); // Pega somente tarefa aberta e concluida
        },

        clean(obj){ //Delete o status do filtro pois preciso somente do nome
            for(var propName in obj){
                if(obj[propName] === null || obj[propName] === undefined){
                    delete obj[propName];
                }
            }
             return obj;
        },

        async clearFilter(){
            this.filter ={
                   subject:null,
                   status:null,
                   
               };

               this.tasks = await this.getTasks();
        },

        overduePresenter(dateOverdue){
            if(!dateOverdue){
                return;
            }    
            return dateOverdue.split('-').reverse().join('/')
        },

        variantOverdue(dateOverdue,taskStatus){
            if(!dateOverdue){
                return 'light';
            }

            if(taskStatus === this.status.FINISHED){
                return 'success';
            }

            let dateNow = new Date().toISOString().split("T")[0];

            if(dateOverdue === dateNow){
                return 'warning';
            }

             if(dateOverdue < dateNow){
                return 'danger';
            }

            return  'light';
        },

        async getTasks(){
            this.isLoading = true;
            let self = this;
            setTimeout(function(){self.isLoading = false}, 1000);
            return await TasksModel.params({
                userId:JSON.parse(localStorage.getItem('authUser')).id,
                status:this.statusList}).get();

        }
    },

    computed:{
        isTasksEmpty(){
            return this.tasks.length == 0
        }
    }
}

</script>


<style scoped>
.empty-data{
    display:flex;
    align-items:center;
    justify-content:center;
    flex-direction: column;
}
.empty-data-image{
    width: 300px;
    height:300px;
}
.finished-task{
    opacity:0.7;
}
/* colocando o estilo de riscar para o corpo do texto tambem */
.finished-task > .card-body > h4, .finished-task > .card-body > p{
    text-decoration: line-through;
} 

.loading-spin{
    display: flex;
    align-items: center;
    justify-content: center;
    height: 65vh;

}
</style>
