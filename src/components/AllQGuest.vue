<template>
    <div>
         <main class="landing-main mw6 m-auto pl15 pr15">
allq

<div class="FAB-button__container mw6 m-auto">
            <router-link :to="{ path: 'my-questions'  }"  class="FAB-button">

			  <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 448 512"><path d="M436 238H242V44c0-6.6-5.4-12-12-12h-12c-6.6 0-12 5.4-12 12v194H12c-6.6 0-12 5.4-12 12v12c0 6.6 5.4 12 12 12h194v194c0 6.6 5.4 12 12 12h12c6.6 0 12-5.4 12-12V274h194c6.6 0 12-5.4 12-12v-12c0-6.6-5.4-12-12-12z"></path></svg>

</router-link>
        </div>
         </main>
         
    </div>
</template>
    

<script>
import { configure } from "radiks";
import Question from "../models/Question";
export default {

    async mounted() {
  
    configure(this.RADIKS_SERVER);

    var q = await Question.fetchList( );


console.log(q);

    $(".question-stats").removeClass("dn");
  }
}
</script>