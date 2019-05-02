<template>
  <div>
    <div class="container">
      <input
        id="searchBar"
        v-model="search"
        class="searchbar active"
        type="text"
        placeholder="Search by github link or owner/reponame"
        @change="init"
      >
      <a id="btnSearch" class="btn-search">
        <i class="fa fa-search"></i>
      </a>
    </div>

    <template v-if="search && !errored">
      <ul class="issues list">
        <li class="issue-item">
          <span>Total number of open issues</span>
          <div class="pull-right">
            <span class="issue-numbers">{{this.finalData.openCount || "..."}}</span>
          </div>
        </li>
        <li class="issue-item">
          <span>Number of open issues that were opened in the last 24 hours</span>
          <div class="pull-right">
            <span class="issue-numbers">{{this.finalData.open24agoCount || "..."}}</span>
          </div>
        </li>
        <li class="issue-item">
          <span>
            Number of open issues that were opened more than 24 hours ago but less
            than 7 days ago
          </span>
          <div class="pull-right">
            <span class="issue-numbers">{{this.finalData.open24agoless7dayCount || "..."}}</span>
          </div>
        </li>
        <li class="issue-item">
          <span>Number of open issues that were opened more than 7 days ago</span>
          <div class="pull-right">
            <span class="issue-numbers">{{this.finalData.openmore7dayCount || "..."}}</span>
          </div>
        </li>
      </ul>
    </template>
    <div v-if="errored" style="color:white;">Please provide a valid link or search term Eg: "https://github.com/vuejs/vue" or "vuejs/vue"</div>
  </div>
</template>

<script>
import axios from "axios";

const GITHUB_TOKEN = process.env.GITHUB_TOKEN;

export default {
  name: "github-search",
  data() {
    return {
      apiUrl: "https://api.github.com/graphql",
      search: "https://github.com/vuejs/vue",
      finalData: {},
      errored: false
    };
  },
  mounted() {
    this.init();
  },
  computed: {
    repoOwnerAndName() {
      let repoUser = "";
      let repoName = "";
      if (this.search) {
        let subUrl = this.search.split("github.com/")[1];
        if (!subUrl) {
          subUrl = this.search;
        }
        repoUser = subUrl.split("/")[0];
        repoName = subUrl.split("/")[1];
      }
      return { repoUser, repoName };
    }
  },
  methods: {
    init() {
      this.finalData = {};
      this.errored = false;
      if (this.search) {
        this.openCountFn();
      }
    },
    dayagoIsoString(ago = 1) {
      return new Date(Date.now() - 86400 * 1000 * ago).toISOString();
    },
    callGithub(query) {
      return axios
        .post(
          this.apiUrl,
          { query: query },
          {
            headers: {
              Authorization: `bearer ${GITHUB_TOKEN}`
            }
          }
        )
        .then(response => {
          return response.data.data;
        })
        .catch(error => {
          return error;
        });
    },
    openCountFn() {
      const query = `query { 
          repository(owner:"${this.repoOwnerAndName.repoUser}", name:"${
        this.repoOwnerAndName.repoName
      }") {
            issues(states:OPEN) {
              totalCount
            }
          }
        }`;
      this.callGithub(query)
        .then(this.open24agoCountFn)
        .catch(error => {
          this.errored = true;
        });
    },
    dayOpenCountFn(since) {
      const query = `query { 
          repository(owner:"${this.repoOwnerAndName.repoUser}", name:"${
        this.repoOwnerAndName.repoName
      }") {
            issues(states: OPEN, filterBy:{since:"${since}"}) {
              totalCount
            }
          }
        }`;
      return this.callGithub(query);
    },
    open24agoCountFn(opencountData) {
      this.$set(
        this.finalData,
        "openCount",
        opencountData.repository.issues.totalCount
      );
      const dayago = this.dayagoIsoString();
      this.dayOpenCountFn(dayago).then(this.open7dayagoCountFn);
    },
    open7dayagoCountFn(opendayagoData) {
      this.$set(
        this.finalData,
        "open24agoCount",
        opendayagoData.repository.issues.totalCount
      );
      const dayago = this.dayagoIsoString(7);
      this.dayOpenCountFn(dayago).then(result => {
        this.$set(
          this.finalData,
          "open24agoless7dayCount",
          result.repository.issues.totalCount - this.finalData.open24agoCount
        );
        this.$set(
          this.finalData,
          "openmore7dayCount",
          this.finalData.openCount - result.repository.issues.totalCount
        );
      });
    }
  }
};
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
.container {
  display: flex;
  flex-direction: row;
  justify-content: center;
  align-items: center;
}
.searchbar {
  float: right;
  background-color: #d03237;
  color: white;
  padding: 6px 10px;
  width: 120px;
  border: none;
  margin-top: 1px;
  margin-right: 8px;
  font-family: "Segoe UI Light", "Segoe UI", "Segoe", Tahoma, Helvetica, Arial,
    sans-serif;
  font-size: 1em;
  font-weight: bold;
  border-bottom: white solid 2px;
  transition: 0.3s;
}
.searchbar::placeholder {
  color: white;
  font-family: "Segoe UI Light", "Segoe UI", "Segoe", Tahoma, Helvetica, Arial,
    sans-serif;
  font-size: 1em;
  font-weight: bold;
  /* transition: 0.2s; */
}
.searchbar.active {
  width: 400px;
  font-family: "Segoe UI Light", "Segoe UI", "Segoe", Tahoma, Helvetica, Arial,
    sans-serif;
  font-size: 1em;
  font-weight: bold;
  transition: 0.3s;
  /* Stops the input box from inheriting the styling from the inputs on the request form */
  border-bottom: white solid 2px;
  outline: none;
}
.btn-search {
  cursor: pointer;
  color: white;
  text-decoration: none !important;
  font-family: "Segoe UI Light", "Segoe UI", "Segoe", Tahoma, Helvetica, Arial,
    sans-serif;
  font-size: 1.5em;
  padding-top: 5px;
  margin-right: 40px;
}
.issues {
  list-style: none;
  margin: 0;
  padding: 0;
  width: 93%;
}
.list .issue-item {
  margin-top: 10px;
  border-radius: 2px;
  background: white;
  box-shadow: 0 2px 1px rgba(140, 134, 134, 0.25);
  position: relative;
  padding: 0 20px;
  font-size: 14px;
  line-height: 40px;
}
.list .issue-item .pull-right {
  position: absolute;
  right: 20px;
  top: 0;
}
@media screen and (max-width: 600px) {
  .list .issue-item .pull-right {
    position: static;
    line-height: 20px;
    padding-bottom: 10px;
  }
}
.issue-numbers {
  color: #a1a1a4;
}
</style>
