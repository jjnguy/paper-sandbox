<script>
  import legislators from "./legislators.json";
  import stateNames from "./stateNames.json";
  import popData from "./nst-est2020.csv";
  import senateVotes from "./1976-2020-senate.csv";

  let senators = legislators.filter(
    (p) => p.terms.filter((t) => t.type == "sen").length > 0
  );

  let repubNos = [
    "Richard Burr",
    "Bill Cassidy",
    "Susan M. Collins",
    "Mitt Romney",
    "Lisa Murkowski",
    "Patrick J. Toomey",
    "Ben Sasse",
  ];

  function votedGuilty(senator) {
    if (senator.terms[0].party != "Republican") return true;
    return repubNos.indexOf(senator.name.official_full) != -1;
  }

  let fullData = senators.map((senator) => {
    let vote = votedGuilty(senator) ? "Guilty" : "Not Guilty";

    let state = {
      code: senator.terms[0].state,
      name: stateNames.filter((s) => s.Code == senator.terms[0].state)[0].State,
      population: popData.filter(
        (row) =>
          row.NAME ==
          stateNames.filter((s) => s.Code == senator.terms[0].state)[0].State
      )[0]?.POPESTIMATE2020,
    };

    let electionData = senateVotes
      .filter(
        (row) =>
          row.candidate.toLowerCase() ==
          senator.name.official_full.toLowerCase()
      )
      .sort(function (one, two) {
        return two.year.localeCompare(one.year);
      })[0];
    return {
      name: senator.name,
      party: senator.terms[0].party,
      vote,
      state,
      electionData: {
        totalVotes: parseInt(electionData?.totalvotes ?? -1),
        candidateVotes: parseInt(electionData?.candidatevotes ?? -1),
        percent: function () {
          return (this.candidateVotes / this.totalVotes) * 100;
        },
      },
    };
  });

  let fullWrapper = {
    tableOps: {
      sortColumn: {
        name: "Name",
        dir: null,
      },
      columns: {
        Name: {
          getValue(senator) {
            return senator.name.official_full;
          },
        },
        State: {
          getValue(senator) {
            return senator.state.name;
          },
        },
        Party: {
          getValue(senator) {
            return senator.party;
          },
        },
        Vote: {
          getValue(senator) {
            return senator.vote;
          },
        },
        "State Pop.": {
          getValue(senator) {
            return senator.state.population;
          },
        },
        "% Vote Received": {
          getValue(senator) {
            return `${senator.electionData.percent().toFixed(2)}%`;
          },
        },
        "Population in Support": {
          getValue(senator) {
            return (
              (senator.electionData.percent() * senator.state.population) /
              100
            ).toFixed(1);
          },
        },
      },
    },
    agregates: {
      "Population in Support": {
        sum() {
          return this.fullData.reduce((result, next) => 6, 0);
        },
      },
    },
    fullData,
  };
</script>

<main>
  <table>
    <thead>
      <tr>
        {#each Object.keys(fullWrapper.tableOps.columns) as col}
          <th on:click={() => (fullWrapper.tableOps.sortColumn.name = col)}>
            {col}
          </th>
        {/each}
      </tr>
    </thead>
    <tbody>
      {#each fullData.sort(function (one, two) {
        let col = fullWrapper.tableOps.columns[fullWrapper.tableOps.sortColumn.name];
        return col.getValue(one).localeCompare(col.getValue(two));
      }) as senator}
        <tr>
          {#each Object.keys(fullWrapper.tableOps.columns) as col}
            <td>
              {fullWrapper.tableOps.columns[col].getValue(senator)}
            </td>
          {/each}
        </tr>
      {/each}
    </tbody>
  </table>
</main>
