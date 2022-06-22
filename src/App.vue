<template>
  <button id="authorize_button" @click="handleAuthClick">Authorize</button>
  <button id="signout_button" @click="handleSignoutClick">Sign Out</button>
  <pre id="content" style="white-space: pre-wrap"></pre>
  <h1>Create Meeting</h1>
  <div>
    <label for="title">Title</label>
    <input type="text" id="title" v-model="title" />
  </div>
  <div>
    <label for="start-date">Start Date</label>
    <input type="date" id="start-date" v-model="startDate" />
  </div>
  <div>
    <label for="start-time">Start Time</label>
    <input type="time" id="start-time" v-model="startTime" />
  </div>
  <div>
    <label for="duration">Duration</label>
    <input type="time" id="duration" v-model="duration" />
  </div>
  <div>
    <label for="frequency">Frequency</label>
    <select id="frequency" v-model="frequency">
      <option value="once">Once</option>
      <option value="daily">Daily</option>
      <option value="weekly">Weekly</option>
    </select>
  </div>
  <div v-if="this.frequency !== 'once'">
    <label for="end-date">End Date</label>
    <input type="date" id="end-date" v-model="endDate" />
  </div>
  <div>
    <label for="attendees">Attendies</label>
    <textarea
      placeholder="Write email address of attendees separated by single comma"
      id="attendees"
      rows="4"
      cols="50"
      v-model="attendees"
    ></textarea>
  </div>
  <div>
    <label for="attendeesVisible">Can attendees see each other?</label>
    <input type="checkbox" id="attendeesVisible" v-model="attendeesVisible" />
  </div>

  <button type="submit" @click="insertEvent">Submit</button>
</template>

<script>
import uniqid from "uniqid";

const CLIENT_ID = "YOUR_CLIENT_ID";
const API_KEY = "YOUR_API_KEY";

// Discovery doc URL for APIs used by the quickstart
const DISCOVERY_DOC =
  "https://www.googleapis.com/discovery/v1/apis/calendar/v3/rest";

// Authorization scopes required by the API; multiple scopes can be
// included, separated by spaces.
const SCOPES = "https://www.googleapis.com/auth/calendar";

export default {
  name: "App",
  data() {
    return {
      title: "",
      startDate: "",
      startTime: "",
      endDate: "",
      finalDate: "",
      finalTime: "",
      duration: "00:00:00",
      frequency: "once",
      attendees: "",
      attendeesVisible: true,
      tokenClient: null,
      gapiInited: false,
      gisInited: false,
      gapi: undefined,
      google: undefined,
    };
  },

  created() {
    this.loadGoogleApi("https://apis.google.com/js/api.js")
      .then(() => {
        this.gapi = window.gapi;
        this.gapiLoaded();
      })
      .catch(() => {
        // Failed to fetch script
      });
    this.loadGoogleApi("https://accounts.google.com/gsi/client").then(() => {
      this.google = window.google;
      this.gisLoaded();
    });
  },

  methods: {
    loadGoogleApi(src) {
      return new Promise(function (resolve, reject) {
        let shouldAppend = false;
        let el = document.querySelector('script[src="' + src + '"]');
        if (!el) {
          el = document.createElement("script");
          el.type = "text/javascript";
          el.async = true;
          el.src = src;
          shouldAppend = true;
        } else if (el.hasAttribute("data-loaded")) {
          resolve(el);
          return;
        }

        el.addEventListener("error", reject);
        el.addEventListener("abort", reject);
        el.addEventListener("load", function loadScriptHandler() {
          el.setAttribute("data-loaded", true);
          resolve(el);
        });

        if (shouldAppend) document.head.appendChild(el);
      });
    },

    gapiLoaded() {
      this.gapi.load("client", this.intializeGapiClient);
    },

    async intializeGapiClient() {
      await this.gapi.client.init({
        apiKey: API_KEY,
        discoveryDocs: [DISCOVERY_DOC],
      });
      this.gapiInited = true;
    },

    gisLoaded() {
      this.tokenClient = this.google.accounts.oauth2.initTokenClient({
        client_id: CLIENT_ID,
        scope: SCOPES,
        callback: "", // defined later
      });
      this.gisInited = true;
    },

    handleAuthClick() {
      this.tokenClient.callback = async (resp) => {
        if (resp.error !== undefined) {
          throw resp;
        }
        document.getElementById("signout_button").style.visibility = "visible";
        document.getElementById("authorize_button").innerText = "Refresh";
        // await this.listUpcomingEvents();
      };

      if (this.gapi.client.getToken() === null) {
        // Prompt the user to select a Google Account and ask for consent to share their data
        // when establishing a new session.
        this.tokenClient.requestAccessToken({ prompt: "consent" });
      } else {
        // Skip display of account chooser and consent dialog for an existing session.
        this.tokenClient.requestAccessToken({ prompt: "" });
      }
    },

    handleSignoutClick() {
      const token = this.gapi.client.getToken();
      if (token !== null) {
        this.google.accounts.oauth2.revoke(token.access_token);
        this.gapi.client.setToken("");
        document.getElementById("content").innerText = "";
        document.getElementById("authorize_button").innerText = "Authorize";
        document.getElementById("signout_button").style.visibility = "hidden";
      }
    },

    getFormattedDate(d) {
      return {
        date: d.getFullYear() + "-" + d.getMonth() + "-" + d.getDate(),
        time: d.getHours() + ":" + d.getMinutes(),
      };
    },

    getSlicedDate(date) {
      const res = [];
      date.split("-").forEach((x) => {
        res.push(parseInt(x));
      });
      return res;
    },

    getSlicedTime(time) {
      const res = [];
      time.split(":").forEach((x) => {
        res.push(parseInt(x));
      });
      return res;
    },

    calculateDate(date, time) {
      const sDate = this.getSlicedDate(date);
      const sTime = this.getSlicedTime(time);
      const d = new Date(
        sDate[0],
        sDate[1],
        sDate[2],
        sTime[0],
        sTime[1],
        0,
        0
      );
      return d;
    },

    calculateFinalTime(date, time, duration) {
      const sDate = this.getSlicedDate(date);
      const sTime = this.getSlicedTime(time);
      const d = new Date(
        sDate[0],
        sDate[1],
        sDate[2],
        sTime[0],
        sTime[1],
        0,
        0
      );
      const diff =
        parseInt(duration.slice(0, 2)) * 60 + parseInt(duration.slice(3));
      const newD = new Date(d.getTime() + diff * 60000);
      this.finalDate =
        newD.getFullYear() + "-" + newD.getMonth() + "-" + newD.getDate();
      this.finalTime = newD.getHours() + ":" + newD.getMinutes();
    },

    calculateDays(initialDate, finalDate) {
      const sDate1 = this.getSlicedDate(initialDate);
      const date1 = new Date(sDate1[0], sDate1[1], sDate1[2], 0, 0, 0, 0);
      const sDate2 = this.getSlicedDate(finalDate);
      const date2 = new Date(sDate2[0], sDate2[1], sDate2[2], 0, 0, 0, 0);
      const difference = date2.getTime() - date1.getTime();
      const TotalDays = Math.ceil(difference / (1000 * 3600 * 24));
      return TotalDays;
    },

    async createMeet({ startDate, startTime, finalDate, finalTime }) {
      let response;
      const event = {
        summary: this.title,
        end: {
          dateTime: `${finalDate}T${finalTime}:00+05:30`,
          timeZone: "UTC",
        },
        start: {
          dateTime: `${startDate}T${startTime}:00+05:30`,
          timeZone: "UTC",
        },
        conferenceData: {
          createRequest: {
            conferenceSolutionKey: {
              type: "hangoutsMeet",
            },
            requestId: uniqid(),
          },
        },
        guestsCanSeeOtherGuests: this.attendeesVisible,
      };
      if (this.attendees) {
        const attendees = [];
        this.attendees
          .split(",")
          .forEach((attendee) => attendees.push({ email: attendee }));
        event.attendees = attendees;
      }
      try {
        response = await this.gapi.client.calendar.events.insert({
          calendarId: "primary",
          resource: event,
          conferenceDataVersion: 1,
        });
        console.log(response.result);
      } catch (error) {
        console.log(error);
      }
    },

    async insertEvent() {
      this.calculateFinalTime(this.startDate, this.startTime, this.duration);
      const starts = [];
      const ends = [];
      const formattedStarts = [];
      const formattedEnds = [];
      starts.push(this.calculateDate(this.startDate, this.startTime));
      ends.push(this.calculateDate(this.finalDate, this.finalTime));
      formattedStarts.push(this.getFormattedDate(starts[0]));
      formattedEnds.push(this.getFormattedDate(ends[0]));

      let count = 0;
      if (this.frequency !== "once") {
        if (this.frequency === "daily") {
          const factor = 1000 * 3600 * 24;
          count = this.calculateDays(this.startDate, this.endDate);
          for (let i = 0; i < count; i++) {
            const newD1 = new Date(
              starts[starts.length - 1].getTime() + factor
            );
            starts.push(newD1);
            formattedStarts.push(
              this.getFormattedDate(starts[starts.length - 1])
            );

            const newD2 = new Date(ends[ends.length - 1].getTime() + factor);
            ends.push(newD2);
            formattedEnds.push(this.getFormattedDate(ends[ends.length - 1]));
          }
        } else {
          count = Math.floor(
            this.calculateDays(this.startDate, this.endDate) / 7
          );
          const factor = 1000 * 3600 * 24 * 7;
          for (let i = 0; i < count; i++) {
            const newD1 = new Date(
              starts[starts.length - 1].getTime() + factor
            );
            starts.push(newD1);
            formattedStarts.push(
              this.getFormattedDate(starts[starts.length - 1])
            );

            const newD2 = new Date(ends[ends.length - 1].getTime() + factor);
            ends.push(newD2);
            formattedEnds.push(this.getFormattedDate(ends[ends.length - 1]));
          }
        }
      }

      console.log(formattedStarts, formattedEnds);

      for (let i = 0; i < 1 + count; i++) {
        await this.createMeet({
          startDate: formattedStarts[i].date,
          startTime: formattedStarts[i].time,
          finalDate: formattedEnds[i].date,
          finalTime: formattedEnds[i].time,
        });
      }
    },
  },
};
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
</style>
