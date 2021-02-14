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
    if (
      senator.terms.sort(function (one, two) {
        return -one.end.localeCompare(two.end);
      })[0].party != "Republican"
    )
      return true;
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
      party: senator.terms.sort(function (one, two) {
        return -one.end.localeCompare(two.end);
      })[0].party,
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
        dir: true,
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
          filterOptions(fullData) {
            return [...new Set(fullData.map((senator) => senator.party))];
          },
          filter: {
            value: null,
          },
        },
        Vote: {
          getValue(senator) {
            return senator.vote;
          },
          filterOptions(fullData) {
            return [...new Set(fullData.map((senator) => senator.vote))];
          },
          filter: {
            value: null,
          },
        },
        "State Pop.": {
          getValue(senator) {
            return senator.state.population;
          },
          sum(filteredData) {
            return (
              filteredData.reduce((result, next) => {
                return parseInt(next.state.population) + result;
              }, 0) / 2
            );
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
          sum(filteredData) {
            return filteredData
              .reduce((result, next) => {
                return (
                  result +
                  (next.electionData.percent() * next.state.population) / 100
                );
              }, 0)
              .toFixed(1);
          },
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
        <th />
        {#each Object.keys(fullWrapper.tableOps.columns) as col}
          <th>
            {col}
            <span
              on:click={() => {
                if (fullWrapper.tableOps.sortColumn.name == col) {
                  fullWrapper.tableOps.sortColumn.dir = !fullWrapper.tableOps
                    .sortColumn.dir;
                } else {
                  fullWrapper.tableOps.sortColumn.name = col;
                  fullWrapper.tableOps.sortColumn.dir = false;
                }
              }}>V</span
            >
            {#if fullWrapper.tableOps.columns[col].filterOptions}
              <select
                bind:value={fullWrapper.tableOps.columns[col].filter.value}
              >
                <option>filter...</option>
                {#each fullWrapper.tableOps.columns[col].filterOptions(fullData) as option}
                  <option value={option}>{option}</option>
                {/each}
              </select>
            {/if}
          </th>
        {/each}
      </tr>
    </thead>
    <tbody>
      {#each fullData
        .filter((senator) => {
          var filters = Object.keys(fullWrapper.tableOps.columns)
            .map((col) => fullWrapper.tableOps.columns[col])
            .filter((col) => col.filter?.value);
          if (filters.length == 0) return true;
          return filters.filter((col) => col.filter.value == col.getValue(senator)).length == filters.length;
        })
        .sort(function (one, two) {
          let col = fullWrapper.tableOps.columns[fullWrapper.tableOps.sortColumn.name];
          return (fullWrapper.tableOps.sortColumn.dir ? 1 : -1) * col
              .getValue(one)
              .localeCompare(col.getValue(two));
        }) as senator, ix}
        <tr>
          <td>{ix + 1}</td>
          {#each Object.keys(fullWrapper.tableOps.columns) as col}
            <td>
              {fullWrapper.tableOps.columns[col].getValue(senator)}
            </td>
          {/each}
        </tr>
      {/each}
      <tr>
        <td>Aggregates</td>
        {#each Object.keys(fullWrapper.tableOps.columns) as col}
          <td>
            {#if fullWrapper.tableOps.columns[col].sum}
              {fullWrapper.tableOps.columns[col].sum(
                fullData.filter((senator) => {
                  var filters = Object.keys(fullWrapper.tableOps.columns)
                    .map((col) => fullWrapper.tableOps.columns[col])
                    .filter((col) => col.filter?.value);
                  if (filters.length == 0) return true;
                  return (
                    filters.filter(
                      (col) => col.filter.value == col.getValue(senator)
                    ).length == filters.length
                  );
                })
              )}
            {/if}
          </td>
        {/each}
      </tr>
    </tbody>
  </table>
</main>
