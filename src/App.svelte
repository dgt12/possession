<script>
  // NB: This code is not optimal; it just demonstrates the concepts
  // TODO: Compartmentalise the code into different components (e.g. one for the sidebar)

  // TODO: Change input data from JS to CSV format
  // TODO: Allow users to upload their own datasets

  // INPUT: Currently expects an array of objects (items) with the same properties (variables)
  // The property values are the categories
  // Datasets: titanic{2}.js, mushrooms{_all}.js, covid.js, crime.js, sleep.js
  import data from "$data/titanic.js";
  // import data from "$data/mushrooms.js";

  // Log raw data
  //console.log("Raw data:");
  //console.log(data);

  // OTHER IMPORTS
  import { scaleOrdinal, scaleLinear } from "d3-scale";
  import { writable, get } from "svelte/store";

  // Initialise processedData as a Svelte store
  const processedData = writable([]);
  // $: if ($processedData) {
  //   console.log("Processed data:", $processedData);
  // }

  // New store to track any selections that have been applied
  const appliedSelections = writable({});
  const selectedCategoriesForQuery = writable({});

  // Initialise categoryCounts
  let categoryCounts = {};

  // The total number of items/records in the full data set
  const totalItems = data.length;

  // Converts the data to frequency form and calculates deviations
  function processData(selectedVariableNames) {
    let combinations = {};
    let categoryCounts = {};
    let totalObservations = 0;

    // Filter data based on selected variables
    let filteredData = data.map((record) => {
      let filteredRecord = {};
      Object.entries(record).forEach(([key, value]) => {
        if (selectedVariableNames.includes(key)) {
          filteredRecord[key] = String(value);
        }
      });
      return filteredRecord;
    });

    // Identify unique combinations and calculate observed frequency
    filteredData.forEach((record) => {
      let key = JSON.stringify(record);
      combinations[key] = (combinations[key] || 0) + 1;
      Object.entries(record).forEach(([category, value]) => {
        categoryCounts[category] = categoryCounts[category] || {};
        categoryCounts[category][value] =
          (categoryCounts[category][value] || 0) + 1;
      });
      totalObservations++;
    });

    // Calculate marginal frequencies
    let marginalFrequencies = {};
    Object.keys(categoryCounts).forEach((category) => {
      marginalFrequencies[category] = {};
      Object.entries(categoryCounts[category]).forEach(([value, count]) => {
        marginalFrequencies[category][value] = count / totalObservations;
      });
    });

    // Compute combinations and their deviations
    let result = Object.entries(combinations).map(([combination, observed]) => {
      let record = JSON.parse(combination);
      let expected = totalObservations;
      Object.entries(record).forEach(([category, value]) => {
        expected *= marginalFrequencies[category][value];
      });
      let deviation = (observed - expected) / Math.sqrt(expected);
      return { combination: record, observed, deviation };
    });

    // Sort the result by frequency and then by deviation
    result.sort((a, b) => b.observed - a.observed || b.deviation - a.deviation);

    // Update processedData store directly
    processedData.set(result);

    // Return only categoryCounts for further processing
    return categoryCounts;
  }

  let originalCategoryCounts = {};
  let isFilterApplied = false;
  let isInitialLoad = true;

  $: if (data && data.length > 0 && isInitialLoad) {
    categoryCounts = processData(selectedVariableNames);
    originalCategoryCounts = { ...categoryCounts };
    const initialSelection = {};
    Object.keys(categoryCounts).forEach((category) => {
      initialSelection[category] = new Set(
        Object.keys(categoryCounts[category])
      );
    });
    selectedCategoriesForQuery.set(initialSelection);
    isInitialLoad = false;
  }

  // Initialise variables from the original data
  let variables = new Set();
  if (data && data.length > 0) {
    data.forEach((item) => {
      Object.keys(item).forEach((key) => {
        variables.add(key);
      });
    });
  }
  variables = Array.from(variables);
  //console.log("Categorical variables:");
  //console.log(variables);

  // Get the initial order of the variables in the input file
  let initialVariableOrder = [];
  $: if (data && data.length > 0 && !initialVariableOrder.length) {
    initialVariableOrder = [...variables];
  }

  let colorScales = {};
  const colorRange = [
    "#a2d4f2",
    "#c4e6a3",
    "#ffb38a",
    "#d8b5eb",
    "#ffdc81",
    "#d4d8de",
  ];
  // Alternative colour scheme (not currently used)
  const colorRange2 = [
    "#63b2ad",
    "#4949c2",
    "#de8a38",
    "#c44b81",
    "#8486f2",
    "#96dd79",
  ];

  // Keep track of which variables to display
  let variableVisibility = {};
  variables.forEach((variable) => {
    variableVisibility[variable] = true; // Display all variables by default
  });

  // Reactive statement to compute selectedVariableNames
  $: selectedVariableNames = Object.keys(variableVisibility).filter(
    (key) => variableVisibility[key]
  );

  function updateVariableVisibility(variable, isVisible) {
    variableVisibility[variable] = isVisible;

    // If the removed variable is the current sort key, reset sorting
    if (!isVisible && variable === sortKey) {
      sortKey = "observed"; // Default to sorting by frequency
      sortOrder = "desc"; // Default to descending order
    }

    // Update selections based on variable visibility
    selectedCategoriesForQuery.update((currentSelections) => {
      const updatedSelections = { ...currentSelections };
      if (!isVisible) {
        // Include all categories of the hidden variable
        updatedSelections[variable] = new Set(
          Object.keys(categoryCounts[variable])
        );
      } else {
        // Re-add variable with all categories selected
        updatedSelections[variable] = new Set(
          Object.keys(categoryCounts[variable])
        );
      }
      return updatedSelections;
    });

    if (isVisible) {
      // Reset the applied selections for the re-added variable
      appliedSelections.update((currentAppliedSelections) => {
        const updatedAppliedSelections = { ...currentAppliedSelections };
        // Remove the re-added variable from applied selections
        delete updatedAppliedSelections[variable];
        return updatedAppliedSelections;
      });
    }

    // Process data and apply sorting based on the current state
    processDataKeepingCurrentState();
  }
  function processDataKeepingCurrentState() {
    const updatedVariableNames = Object.keys(variableVisibility).filter(
      (key) => variableVisibility[key]
    );

    // Process data with updated variables
    categoryCounts = processData(updatedVariableNames);

    // Update color scales based on the current state of categoryCounts
    updateColorScales();

    // Reapply sorting logic based on the current sort state
    applySortingBasedOnState();
  }

  // Reactive statement to update processedData and categoryCounts
  $: if (selectedVariableNames.length > 0) {
    // console.log(
    //   "[Reactive Statement] selectedVariableNames changed:",
    //   selectedVariableNames
    // );
    categoryCounts = processData(selectedVariableNames);
    updateColorScales(); // Call function to update color scales
  } else {
    processedData.set([]);
  }

  // Function to update color scales based on current categoryCounts
  function updateColorScales() {
    Object.keys(categoryCounts).forEach((category) => {
      let sortedCategories = Object.entries(categoryCounts[category]);

      // Check if the category is ordinal (starts with a numeral)
      if (sortedCategories.length > 0 && /^\d/.test(sortedCategories[0][0])) {
        // Sort categories numerically
        sortedCategories.sort((a, b) => {
          return parseInt(a[0], 10) - parseInt(b[0], 10);
        });
        // Apply greyscale
        const greyScale = scaleLinear()
          .domain([0, sortedCategories.length - 1])
          .range(["#f0f0f0", "#303030"]); // Light grey to dark grey
        colorScales[category] = scaleOrdinal()
          .domain(sortedCategories.map((entry) => entry[0]))
          .range(sortedCategories.map((entry, index) => greyScale(index)));
      } else {
        // For nominal variables, use existing color logic
        sortedCategories.sort((a, b) => b[1] - a[1]);
        let sortedCategoryNames = sortedCategories.map((entry) => entry[0]);
        if (sortedCategoryNames.length > 5) {
          let extendedRange = new Array(sortedCategoryNames.length).fill(
            colorRange[colorRange.length - 1]
          );
          colorRange.slice(0, 5).forEach((color, index) => {
            extendedRange[index] = color;
          });
          colorScales[category] = scaleOrdinal()
            .domain(sortedCategoryNames)
            .range(extendedRange);
        } else {
          colorScales[category] = scaleOrdinal()
            .domain(sortedCategoryNames)
            .range(colorRange);
        }
      }
    });
  }

  // Determines whether the given variable is ordinal
  function isOrdinal(variable) {
    if (categoryCounts[variable]) {
      // Check if any category of this variable starts with a digit
      return Object.keys(categoryCounts[variable]).some((category) => {
        return !isNaN(category.charAt(0));
      });
    }
    return false; // Return false if the variable is not found or has no categories starting with a digit
  }

  // Give ordinal categories with larger values (upper half) white text to improve readability
  function getStickerClass(category, property) {
    if (isOrdinal(property)) {
      // Extract numeric values from categories, sort them numerically
      const sortedCategories = Object.keys(categoryCounts[property])
        .map(extractLeadingNumber)
        .filter((num) => num !== null)
        .sort((a, b) => a - b);

      const categoryNumber = extractLeadingNumber(category);
      const midpoint = Math.ceil(sortedCategories.length / 2);
      const categoryIndex = sortedCategories.indexOf(categoryNumber);

      if (categoryIndex >= midpoint) {
        return "sticker sticker-dark";
      }
    }
    return "sticker";
  }

  // COMPUTE BAR WIDTHS FOR MAIN VIS
  // Calculate width for frequency column
  const cell_width = 80;
  // Find the largest frequency
  $: maxFrequency = Math.max(...$processedData.map((item) => item.observed));
  // Scale such that the largest frequency occupies the entire column width
  $: frequencyScaleFactor = cell_width / maxFrequency;

  // Calculate width for deviation column
  // Find the maximum (absolute) deviation
  $: maxDeviation = Math.max(
    ...$processedData.map((item) => Math.abs(item.deviation))
  );
  // Scale such that the largest bar occupies half the column width
  // (allocate same amount of space for negative and positive values)
  $: deviationScaleFactor = cell_width / 2 / maxDeviation;

  // SORT INDIVIDUAL COLS
  let sortKey = "observed";
  // Initial sort order
  let sortOrder = "desc";
  // Tracks if a column is being dragged
  let isDragging = false;

  // Separate function for sorting by query
  function sortByQuery() {
    processedData.update((data) => {
      data.sort((a, b) => {
        const matchA = doesRowMatchQuery(a);
        const matchB = doesRowMatchQuery(b);
        if (matchA && !matchB) return -1;
        if (!matchA && matchB) return 1;
        return 0;
      });
      return data;
    });
  }

  // Sort by column
  function sortByColumns() {
    processedData.update((data) => {
      data.sort((a, b) => {
        let primarySortResult = 0;
        if (sortKey === "observed" || sortKey === "deviation") {
          primarySortResult =
            sortOrder === "desc"
              ? b[sortKey] - a[sortKey]
              : a[sortKey] - b[sortKey];
        } else if (isOrdinal(sortKey)) {
          const numA = extractLeadingNumber(a.combination[sortKey]);
          const numB = extractLeadingNumber(b.combination[sortKey]);
          if (numA !== null && numB !== null) {
            primarySortResult =
              sortOrder === "desc" ? numB - numA : numA - numB;
          }
        } else {
          // Sort based on categoryCounts for categorical columns
          const frequencyA = categoryCounts[sortKey]
            ? categoryCounts[sortKey][a.combination[sortKey]] || 0
            : 0;
          const frequencyB = categoryCounts[sortKey]
            ? categoryCounts[sortKey][b.combination[sortKey]] || 0
            : 0;
          primarySortResult =
            sortOrder === "desc"
              ? frequencyB - frequencyA
              : frequencyA - frequencyB;
          // Secondary sorting by category name if frequencies are equal
          if (primarySortResult === 0) {
            return a.combination[sortKey].localeCompare(
              b.combination[sortKey],
              undefined,
              { numeric: true }
            );
          }
        }
        // Secondary sort by deviation if there's a tie in primary sort and not sorting by deviation
        if (primarySortResult === 0 && sortKey !== "deviation") {
          return b.deviation - a.deviation; // Always descending for deviation
        }
        return primarySortResult;
      });
      return data;
    });
  }
  // Update the sortData function
  function sortData(column) {
    if (isDragging) {
      console.log("Column is being dragged, not sorting.");
      return;
    }
    if (column && column !== "default") {
      if (sortKey === column) {
        sortOrder = sortOrder === "desc" ? "asc" : "desc";
      } else {
        sortKey = column;
        sortOrder = "desc";
      }
    } else {
      // Set default sortKey to 'observed' and sortOrder to 'desc' for initial load
      sortKey = sortKey || "observed";
      sortOrder = sortOrder || "desc";
    }
    // Apply column sorting first
    sortByColumns();
    // Then apply query sorting if checkbox is checked
    if (sortByQueryFirst) {
      sortByQuery();
    }
  }
  // Reactive statement for immediate reactivity on query changes
  $: if ($selectedCategoriesForQuery) {
    if (sortByQueryFirst) {
      sortByQuery();
    } else {
      sortByColumns(); // Apply column sorting when query changes but checkbox is unchecked
    }
  }

  $: if (sortByQueryFirst) {
    sortByQuery();
  } else {
    sortByColumns();
  }

  // ALLOW DRAGGING OF VARIABLE COLUMNS
  let draggedCol = null;
  let overCol = null;

  function onDragStart(event, columnName) {
    console.log(`Dragging: ${columnName}`);
    isDragging = true;
    draggedCol = columnName;
    event.dataTransfer.effectAllowed = "move";
  }

  function onDragOver(event, columnName) {
    event.preventDefault(); // Necessary to allow dropping
    console.log(`Drag over: ${columnName}`);
    overCol = columnName;
  }

  function onDrop(event) {
    event.preventDefault();
    console.log(`Dropped on: ${overCol}`);

    if (draggedCol && overCol && draggedCol !== overCol) {
      // Reorder columns
      let draggedIndex = variables.findIndex((p) => p === draggedCol);
      let overIndex = variables.findIndex((p) => p === overCol);

      variables.splice(overIndex, 0, variables.splice(draggedIndex, 1)[0]);
      variables = [...variables]; // Trigger reactivity by creating a new array
      console.log("After reorder:", variables);
    }

    // Reset drag state
    resetDragState();
  }

  function onDragEnd() {
    console.log(`Drag ended`);
    resetDragState();
  }

  function resetDragState() {
    isDragging = false;
    draggedCol = null;
    overCol = null;
  }

  // Sorting categories by frequency
  let sortedCategories = {};

  // Function to extract the leading number from a category name
  function extractLeadingNumber(str) {
    const match = str.match(/^(\d+)/);
    return match ? parseInt(match[1], 10) : null;
  }

  $: {
    if (categoryCounts) {
      Object.keys(categoryCounts).forEach((property) => {
        if (isOrdinal(property)) {
          sortedCategories[property] = Object.keys(
            categoryCounts[property]
          ).sort((a, b) => {
            const numA = extractLeadingNumber(a);
            const numB = extractLeadingNumber(b);
            if (numA !== null && numB !== null) {
              // Compare as numbers if both have leading numbers
              return numA - numB;
            } else {
              // Fallback to string comparison
              return a.localeCompare(b);
            }
          });
        } else {
          // Existing logic for non-ordinal variables
          sortedCategories[property] = Object.entries(categoryCounts[property])
            .sort((a, b) => b[1] - a[1])
            .map((entry) => entry[0]);
        }
      });
    }
  }
  // console.log("Sorted categories:");
  // console.log(sortedCategories);

  // SIDEBAR HELPER FUNCTIONS
  // Function to calculate bar width for the sidebar
  function getGlobalMaxFrequency(categoryCounts) {
    let maxFrequency = 0;
    // For each variable
    Object.values(categoryCounts).forEach((variable) => {
      // Get the category with the maximum frequency
      const maxCatFrequency = Math.max(...Object.values(variable));
      // If the current variable's max frequency is the highest seen so far, update it to be the new maximum freq
      if (maxCatFrequency > maxFrequency) {
        maxFrequency = maxCatFrequency;
      }
    });
    return maxFrequency;
  }

  $: globalMaxFrequency = getGlobalMaxFrequency(categoryCounts);
  // $: console.log({ categoryCounts });

  let viewMode = "standard"; // Default to standard view

  $: if (viewMode) {
    updateCategoryBars();
  }

  function updateCategoryBars() {
    // Trigger a re-render or recalculation that affects the category bars
    // For instance, you can update a variable that is used in the rendering of the bars
    categoryCheckboxes = Object.assign({}, categoryCheckboxes); // Triggers reactivity
  }

  function calculateBarWidth(property, category) {
    const categoryFrequency =
      categoryCounts[property] && categoryCounts[property][category]
        ? categoryCounts[property][category]
        : 0;

    if (viewMode === "normalised") {
      return 100;
    } else {
      // Standard view
      return (categoryFrequency / globalMaxFrequency) * 100;
    }
  }

  // SUMMARY STATISTICS
  let selectedItems = 0;
  $: {
    selectedItems = $processedData.reduce((sum, item, index) => {
      return sum + (matchedRows[index] ? item.observed : 0);
    }, 0);
  }
  //console.log("Selected items:");
  //console.log(selectedItems);

  $: nonfilteredItems = $processedData.reduce(
    (acc, item) => acc + item.observed,
    0
  );

  $: selectedPercentage =
    nonfilteredItems > 0 ? (selectedItems / nonfilteredItems) * 100 : 0;

  $: variablesShown = Object.values(variableVisibility).filter(
    (isVisible) => isVisible
  ).length; // This will be the number of combination properties in the filtered array

  // Assuming the first item is a complete record
  const totalVariables =
    data && data.length > 0 ? Object.keys(data[0]).length : 0;
  // console.log("total variables:");
  // console.log(totalVariables);

  // Variable for the number of rows currently selected in the table
  $: selectedRows = matchedRows.filter((isMatch) => isMatch).length;
  // Variable for the total number of observed combinations for the given variables
  $: totalRows = $processedData.length || 0.00001;

  //$: console.log("Selected rows:", selectedRows);

  // Function to select only one category and deselect others
  function selectOnlyThisCategory(variable, selectedCategory) {
    selectedCategoriesForQuery.update((currentSelection) => {
      const newSelection = new Set();
      newSelection.add(selectedCategory);
      return { ...currentSelection, [variable]: newSelection };
    });
    // Apply sorting logic
    applySortingBasedOnState();
  }

  // Function to apply sorting logic based on the state of sortByQueryFirst
  function applySortingBasedOnState() {
    if (sortByQueryFirst) {
      sortByQuery();
    } else {
      sortByColumns();
    }

    // Reapply the sorting logic after updating the query
    if (sortByQueryFirst) {
      sortByQuery();
    }
    sortByColumns();
  }

  $: matchedRows = $processedData.map((dataItem) =>
    doesRowMatchQuery(dataItem)
  );
  // $: console.log("Matched rows:", { matchedRows });

  function doesRowMatchQuery(dataItem) {
    let query = $selectedCategoriesForQuery; // Access the store's value
    return Object.entries(query).every(([variable, selectedCategories]) => {
      // Skip the variables that are not visible
      if (!variableVisibility[variable]) return true;

      if (!dataItem.combination[variable]) return false;
      return selectedCategories.has(dataItem.combination[variable]);
    });
  }

  let sortByQueryFirst = true;

  // Reactive statement to track checkbox states for each category
  let categoryCheckboxes = {};
  $: {
    categoryCheckboxes = {};
    const selection = $selectedCategoriesForQuery;
    variables.forEach((variable) => {
      categoryCheckboxes[variable] = sortedCategories[variable].map(
        (category) => ({
          category,
          isChecked: selection[variable]?.has(category),
        })
      );
    });
  }

  // Function to handle checkbox change
  function handleCheckboxChange(variable, category, isChecked) {
    selectedCategoriesForQuery.update((currentSelection) => {
      const updatedSelection = new Set(currentSelection[variable] || []);
      if (isChecked) {
        updatedSelection.add(category);
      } else {
        updatedSelection.delete(category);
      }
      return { ...currentSelection, [variable]: updatedSelection };
    });
    applySortingBasedOnState();
  }

  // Hover effect for category bars
  let hoveredCategory = null;
  let hoveredProperty = null;

  function handleMouseEnter(property, category) {
    hoveredCategory = category;
    hoveredProperty = property;
  }

  function handleMouseLeave() {
    hoveredCategory = null;
    hoveredProperty = null;
  }

  // Add the toggleCategorySelection call inside the script tag
  // Function to toggle category selection
  function toggleCategoryLabel(property, category) {
    // Find the checkbox corresponding to the category
    const checkbox = document.querySelector(
      `#checkbox-${property}-${category}`
    );
    if (checkbox) {
      checkbox.click(); // Programmatically click the checkbox
    }
  }

  // Function to calculate the selected frequency for a category
  function calculateSelectedFrequency(property, category) {
    let selectedFrequency = 0;
    $processedData.forEach((dataItem, index) => {
      if (matchedRows[index] && dataItem.combination[property] === category) {
        selectedFrequency += dataItem.observed;
      }
    });
    return selectedFrequency;
  }

  // Function to calculate the percentage of selected frequency out of the total frequency for a category
  function calculateSelectedPercentage(property, category) {
    const selectedFrequency = calculateSelectedFrequency(property, category);
    const totalCategoryFrequency = categoryCounts[property][category];
    return totalCategoryFrequency > 0
      ? (selectedFrequency / totalCategoryFrequency) * 100
      : 0;
  }

  // Function to calculate the width of the selected segment
  function calculateSelectedBarWidth(property, category) {
    const selectedFrequency = calculateSelectedFrequency(property, category);
    const totalCategoryFrequency = categoryCounts[property][category];
    return totalCategoryFrequency > 0
      ? (selectedFrequency / totalCategoryFrequency) * 100
      : 0;
  }

  function resetDisplay() {
    // Reset variable visibility to true for all variables
    variableVisibility = {};
    variables.forEach((variable) => {
      variableVisibility[variable] = true;
    });

    // Reset the order of variables to the initial order
    variables = [...initialVariableOrder];

    // Reset selected categories for query to include all categories
    const resetSelection = {};
    Object.keys(originalCategoryCounts).forEach((category) => {
      resetSelection[category] = new Set(
        Object.keys(originalCategoryCounts[category])
      );
    });
    selectedCategoriesForQuery.set(resetSelection);

    // Reset the applied selections
    appliedSelections.set({});

    viewMode = "standard";

    // Reset sorting options
    sortKey = "observed";
    sortOrder = "desc"; // Initial sort order

    // Trigger data processing with all variables visible
    categoryCounts = processData(variables);

    // Scroll to the top of the sidebar
    const sidebar = document.querySelector(".sidebar");
    if (sidebar) {
      sidebar.scrollTop = 0;
    }

    // Scroll to the top of the main window
    window.scrollTo(0, 0);
  }

  // Determines whether category is missing data
  function isSpecialCategory(category) {
    const lowerCaseCategory = category.toLowerCase();
    const specialCategories = ["unknown", "missing", "?", "n/a"];
    return specialCategories.includes(lowerCaseCategory);
  }

  // FILTERING
  // Function to filter data by selection
  function filterBySelection() {
    const currentData = get(processedData);
    const selectedData = currentData.filter((_, index) => matchedRows[index]);

    if (selectedData.length > 0) {
      processedData.set(selectedData);

      // Recalculate category frequencies for filtered data
      recalculateCategoryFrequencies(selectedData);

      // Update nonfilteredItems
      nonfilteredItems = selectedData.reduce(
        (sum, item) => sum + item.observed,
        0
      );

      // Set applied selections to the current selections
      appliedSelections.set(get(selectedCategoriesForQuery));

      // Mark that filtering has been applied
      isFilterApplied = true;
    }
  }

  // Adjust the category label styling logic
  // Check if the category should be bold based on applied selections
  function isBoldCategory(property, category) {
    let appliedSelection = get(appliedSelections)[property];
    // console.log(
    //   `[isBoldCategory] Checking if ${category} in ${property} is bold. Applied selection:`,
    //   appliedSelection
    // );
    if (!appliedSelection) return false;

    // Check if the number of selected categories is less than the original number of categories
    const originalCategoryCount = originalCategoryCounts[property]
      ? Object.keys(originalCategoryCounts[property]).length
      : 0;
    return (
      appliedSelection.size < originalCategoryCount &&
      appliedSelection.has(category)
    );
  }

  function recalculateCategoryFrequencies(filteredData) {
    let newCategoryCounts = {};
    filteredData.forEach((dataItem) => {
      Object.entries(dataItem.combination).forEach(([category, value]) => {
        newCategoryCounts[category] = newCategoryCounts[category] || {};
        newCategoryCounts[category][value] =
          (newCategoryCounts[category][value] || 0) + dataItem.observed;
      });
    });
    categoryCounts = newCategoryCounts; // Update the global variable
  }

  $: maxFrequencyWidth = getMaxFrequencyWidth($processedData);

  function getMaxFrequencyWidth(data) {
    if (data.length === 0) return 0;

    const maxFrequency = Math.max(...data.map((item) => item.observed));
    const numberLength = maxFrequency.toLocaleString().length;
    const scalingFactor = 10; // Adjust this based on font size and desired spacing

    return numberLength * scalingFactor;
  }

  let isButtonEnabled = true;

  $: {
    isButtonEnabled = selectedPercentage > 0 && selectedPercentage < 100;
  }

  let tooltipText = "";

  $: {
    if (selectedPercentage === 0) {
      tooltipText = "No items match the current selection."; // Please ensure at least one category has been selected for each active variable.";
    } else if (selectedPercentage === 100) {
      tooltipText =
        "You must select a subset of categories to use this feature.";
    } else {
      tooltipText = ""; // No tooltip for other cases
    }
  }

  // QUERY VIA STICKERS
  // When a sticker is clicked, the function first retrieves the current selection for that variable.
  function handleStickerClick(variable, category) {
    selectedCategoriesForQuery.update((currentSelection) => {
      const updatedSelection = new Set(currentSelection[variable] || []);

      // If all categories are selected or none are selected, select only the clicked category
      if (
        updatedSelection.size ===
          Object.keys(categoryCounts[variable]).length ||
        updatedSelection.size === 0
      ) {
        updatedSelection.clear();
        updatedSelection.add(category);
      } else {
        // If some categories are selected, toggle the clicked category
        if (updatedSelection.has(category)) {
          updatedSelection.delete(category);
        } else {
          updatedSelection.add(category);
        }
      }

      return { ...currentSelection, [variable]: updatedSelection };
    });

    applySortingBasedOnState();
  }
</script>

<main>
  <table>
    <thead>
      <tr>
        {#each variables as property}
          {#if variableVisibility[property]}
            <th
              draggable="true"
              on:dragstart={(event) => onDragStart(event, property)}
              on:dragover={(event) => onDragOver(event, property)}
              on:drop={onDrop}
              on:dragend={onDragEnd}
              on:click={() => sortData(property)}
              title={`${property} ${
                sortKey === property ? (sortOrder === "asc" ? "↑" : "↓") : ""
              }`.trim()}
            >
              <div class="heading-container">
                <span class="heading">{property}</span>
                {#if sortKey === property}
                  <svg class="sort-icon" viewBox="0 0 10 10" aria-hidden="true">
                    <polygon
                      points={sortOrder === "asc"
                        ? "0,10 5,0 10,10"
                        : "0,0 5,10 10,0"}
                      class="full-opacity"
                    />
                  </svg>
                {:else}
                  <svg class="sort-icon" viewBox="0 0 10 10" aria-hidden="true">
                    <polygon points="0,0 5,10 10,0" class="reduced" />
                  </svg>
                {/if}
              </div></th
            >
          {/if}
        {/each}
        {#if selectedVariableNames.length > 0}
          <th
            on:click={() => sortData("observed")}
            class="frequency-heading-container"
            style="min-width: {maxFrequencyWidth}px;"
            title={`Frequency${
              sortKey === "observed"
                ? sortOrder === "asc"
                  ? " (↑)"
                  : " (↓)"
                : ""
            }: The number of observations with the specified combination of categories`.trim()}
          >
            <span class="heading-container numeric">
              <span class="heading">Frequency</span>
              <!-- Add SVG Triangle -->
              {#if sortKey === "observed"}
                <svg class="sort-icon" viewBox="0 0 10 10" aria-hidden="true">
                  <polygon
                    points={sortOrder === "asc"
                      ? "0,10 5,0 10,10"
                      : "0,0 5,10 10,0"}
                    class="full-opacity"
                  />
                </svg>
              {:else}
                <svg class="sort-icon" viewBox="0 0 10 10" aria-hidden="true">
                  <polygon points="0,0 5,10 10,0" class="reduced" />
                </svg>
              {/if}
            </span>
          </th>
          <th
            on:click={() => sortData("deviation")}
            title={`Residual${
              sortKey === "deviation"
                ? sortOrder === "asc"
                  ? " (↑)"
                  : " (↓)"
                : ""
            }: The deviation of the given combination's observed frequency from its expected frequency (red=under-represented, blue=over-represented)`.trim()}
          >
            <span class="heading-container numeric">
              <span class="heading">Residual</span>
              <!-- Add SVG Triangle -->
              {#if sortKey === "deviation"}
                <svg class="sort-icon" viewBox="0 0 10 10" aria-hidden="true">
                  <polygon
                    points={sortOrder === "asc"
                      ? "0,10 5,0 10,10"
                      : "0,0 5,10 10,0"}
                    class="full-opacity"
                  />
                </svg>
              {:else}
                <svg class="sort-icon" viewBox="0 0 10 10" aria-hidden="true">
                  <polygon points="0,0 5,10 10,0" class="reduced" />
                </svg>
              {/if}
            </span>
          </th>
        {/if}
      </tr>
    </thead>
    <tbody>
      {#each $processedData as dataItem, index}
        <tr class={matchedRows[index] ? "full-opacity" : "reduced-opacity"}>
          {#each variables as property}
            {#if variableVisibility[property]}
              <td>
                {#if property in dataItem.combination}
                  <span
                    class={getStickerClass(
                      dataItem.combination[property],
                      property
                    )}
                    title={dataItem.combination[property]}
                    style="background-color: {typeof colorScales[property] ===
                    'function'
                      ? colorScales[property](dataItem.combination[property])
                      : 'blue'}"
                    on:click={() =>
                      handleStickerClick(
                        property,
                        dataItem.combination[property]
                      )}
                  >
                    {dataItem.combination[property]}
                  </span>
                {/if}
              </td>
            {/if}
          {/each}
          <td
            title={`Frequency: ${dataItem.observed.toLocaleString()}/${nonfilteredItems.toLocaleString()} (${(
              (dataItem.observed / nonfilteredItems) *
              100
            ).toFixed(0)}%)`}
          >
            <div class="frequency-bar-container">
              <div
                class="bar freq"
                style="width: {dataItem.observed *
                  frequencyScaleFactor}px; background-color: gold; height: 20px;"
              ></div>
              <span
                class="frequency-label"
                style="margin-left: {dataItem.observed * frequencyScaleFactor +
                  12}px">{dataItem.observed.toLocaleString()}</span
              >
            </div>
          </td>
          <td title={`Residual: ${dataItem.deviation.toFixed(2)}`}>
            <div class="deviation-container">
              {#if dataItem.deviation > 0}
                <div
                  class="bar positive"
                  style="width: {dataItem.deviation * deviationScaleFactor}px;"
                />
              {:else if dataItem.deviation < 0}
                <div
                  class="bar negative"
                  style="width: {Math.abs(dataItem.deviation) *
                    deviationScaleFactor}px;"
                />
              {/if}
            </div>
          </td>
        </tr>
      {/each}
    </tbody>
  </table>
  <!-- Sidebar Markup -->
  <div class="sidebar">
    <div class="selected-data">
      <div class="selected-text">
        Selected items: {selectedItems.toLocaleString()} ({selectedPercentage.toFixed(
          0
        )}%)
      </div>
      <div
        class="progress-bar-container"
        title={`Selected items: ${selectedItems.toLocaleString()}/${nonfilteredItems.toLocaleString()} (${selectedPercentage.toFixed(
          0
        )}%)`}
      >
        <div
          class="progress-bar-selected"
          style="width: {selectedPercentage}%;"
        />
        <div
          class="progress-bar-nonselected"
          style="width: {100 - selectedPercentage}%;"
        />
      </div>
      <!-- Summary Statistics Section -->
      <div class="summary-statistics">
        <div class="statistic">
          <span>Items considered:</span>
          <span
            >{nonfilteredItems.toLocaleString()} ({(
              (nonfilteredItems / totalItems) *
              100
            ).toFixed(0)}%)</span
          >
        </div>
        <div class="statistic">
          <span>Selected rows:</span>
          <span
            >{selectedRows} ({((selectedRows / totalRows) * 100).toFixed(
              0
            )}%)</span
          >
        </div>
        <div class="statistic">
          <span>Variables shown:</span>
          <span
            >{variablesShown} ({(
              (variablesShown / totalVariables) *
              100
            ).toFixed(0)}%)</span
          >
        </div>
        <div class="sort-by-query-checkbox">
          <input
            type="checkbox"
            id="sort-by-query-checkbox"
            bind:checked={sortByQueryFirst}
          />
          <label for="sort-by-query-checkbox">List selected rows first</label>
          <div class="view-mode-selector">
            <label>
              <input
                type="radio"
                bind:group={viewMode}
                value="standard"
                checked
              />
              Standard
            </label>
            <label>
              <input type="radio" bind:group={viewMode} value="normalised" />
              100% Bars
            </label>
          </div>
        </div>
      </div>
    </div>
    <div class="category-bars">
      {#each variables as property}
        <div class="variable-box">
          <!-- Checkbox for each variable (property) -->
          <span class="var-tickbox">
            <input
              type="checkbox"
              id={`checkbox-${property}`}
              checked={variableVisibility[property]}
              on:change={() =>
                updateVariableVisibility(
                  property,
                  !variableVisibility[property]
                )}
            />
            <label for={`checkbox-${property}`}>{property}</label>
          </span>

          {#if variableVisibility[property]}
            {#each categoryCheckboxes[property] as { category, isChecked }}
              <div class="category-container">
                <span
                  class="category-label {isSpecialCategory(category)
                    ? 'special-category-label'
                    : ''} {isBoldCategory(property, category)
                    ? 'bold-category-label'
                    : ''}"
                  title={`${category}: ${calculateSelectedFrequency(
                    property,
                    category
                  ).toLocaleString()}/${categoryCounts[property][
                    category
                  ].toLocaleString()} (${calculateSelectedPercentage(
                    property,
                    category
                  ).toFixed(0)}%)`}
                  on:click={() => toggleCategoryLabel(property, category)}
                  >{category}</span
                >
                <input
                  type="checkbox"
                  class="category-checkbox"
                  id={`checkbox-${property}-${category}`}
                  checked={isChecked}
                  on:change={(event) =>
                    handleCheckboxChange(
                      property,
                      category,
                      event.target.checked
                    )}
                />
                <div
                  class="category-clickable-area"
                  on:mouseenter={() => handleMouseEnter(property, category)}
                  on:mouseleave={handleMouseLeave}
                  on:click={() => selectOnlyThisCategory(property, category)}
                  title={`${category}: ${calculateSelectedFrequency(
                    property,
                    category
                  ).toLocaleString()}/${categoryCounts[property][
                    category
                  ].toLocaleString()} (${calculateSelectedPercentage(
                    property,
                    category
                  ).toFixed(0)}%)`}
                >
                  <div
                    class="category-bar-container"
                    style="width: {calculateBarWidth(property, category)}%;"
                  >
                    <div
                      class="category-bar-selected"
                      style="width: {calculateSelectedBarWidth(
                        property,
                        category
                      )}%; background-color: {colorScales[property](category)};"
                    ></div>
                    <div
                      class="category-bar-unselected"
                      style="width: {100 -
                        calculateSelectedBarWidth(
                          property,
                          category
                        )}%; background-color: {colorScales[property](
                        category
                      )}; opacity: 0.3;"
                    ></div>
                  </div>
                </div>
              </div>
            {/each}
          {/if}
        </div>
      {/each}
    </div>
    <!-- Filter Button -->
    <div class="buttons">
      <button
        on:click={filterBySelection}
        class="filter {isButtonEnabled ? '' : 'disabled'}"
        disabled={!isButtonEnabled}
        title={isButtonEnabled ? "" : tooltipText}
      >
        Filter by selection</button
      >
      <button on:click={resetDisplay} class="reset">Reset</button>
    </div>
  </div>
</main>

<style>
  /*TODO: Refactor all CSS rules*/
  .category-bars {
    margin-left: 0%;
  }
  table {
    border-collapse: collapse;
    overflow: scroll;
    /*max-width: 70vw;*/
    /*width: 75vw;*/
  }
  thead {
    /*font-size: 0.95rem;*/
    background-color: white;
    /*background-color: #eee;*/
    left: 0;
    user-select: none;
    position: -webkit-sticky;
    position: sticky;
    top: 0;
    z-index: 1;
  }
  .sort-by-query-checkbox {
    font-size: 0.9rem;
    margin-left: -4px;
    user-select: none;
  }
  .reduced {
    opacity: 0.3;
  }
  .reduced-opacity {
    /* Style for non-matching rows in the table */
    opacity: 0.3;
  }
  .full-opacity {
    /* Style for matching rows in the table */
    opacity: 1;
  }
  .sort-icon {
    width: 9px; /* Adjust size as needed */
    height: 9px; /* Adjust size as needed */
    fill: currentColor; /* Use the current text color */
    opacity: 0.8; /* Default opacity for inactive sort icons */
  }
  .summary-statistics {
    display: flex; /* Make this a flex container */
    justify-content: center; /* Center content horizontally */
    flex-direction: column; /* Stack children vertically */
    align-items: flex-start; /* Align children (statistics) to the start (left) */
    margin-left: auto; /* Center the entire block horizontally */
    margin-right: auto;
    width: fit-content; /* Width should fit the content */
    margin-bottom: 20px;
    margin-top: 12px;
  }
  .summary-statistics .statistic {
    text-align: left; /* Align text to the left */
    margin-bottom: 10px;
    font-size: 0.9rem;
    user-select: none;
  }
  .selected-text {
    font-size: 0.95rem;
    user-select: none;
    font-weight: 600;
    padding-top: 3px;
    padding-bottom: 8px;
  }
  .variable-box {
    padding-bottom: 10px;
    text-align: left;
    user-select: none;
    /*cursor: pointer;*/
  }
  .var-tickbox {
    margin-left: 38%;
    /*cursor: pointer;*/
  }
  .variable-box:last-child {
    margin-bottom: 10px; /* Adjust this value as needed */
  }
  .category-container {
    display: flex;
    cursor: default;
    align-items: center;
    height: 18px;
    margin-top: 4px;
    margin-bottom: 4px;
  }
  .category-clickable-area:hover .category-bar-container {
    border: 1px solid black; /* Black outline on hover */
  }
  .category-bar-selected,
  .category-bar-unselected {
    height: 100%;
  }
  .category-bar-container {
    display: flex;
    align-items: center;
    height: 17px; /* or another appropriate height */
  }
  .category-checkbox {
    flex-shrink: 0;
    margin-left: -2px;
    margin-right: 8px;
    cursor: pointer;
  }
  .category-label,
  .category-checkbox {
    flex-shrink: 0; /* Prevent these elements from shrinking */
  }
  .category-clickable-area {
    flex-grow: 1; /* Fill the remaining space */
    display: flex;
    align-items: center;
    cursor: pointer;
  }
  .category-label {
    cursor: pointer;
    width: 30%; /* Fixed width for labels */
    text-align: right; /* Right-align the text */
    flex-shrink: 0; /* Prevent the label from shrinking */
    overflow: hidden; /* Hide overflow */
    white-space: nowrap; /* Keep text on a single line */
    text-overflow: ellipsis; /* Add ellipsis for overflowed text */
    margin-right: 8px; /* Space between label and bar */
    line-height: 1.2; /* Increase line height */
    padding-top: 2px; /* Slight padding at the top */
    padding-bottom: 2px; /* Slight padding at the bottom */
  }
  .selected-data {
    display: block; /* This makes each element take its own line */
    text-align: center;
    margin-bottom: 20px; /* Adds some space below each element */
    /*margin-left: 25%;*/
    position: sticky;
    top: 0; /* Stick to the top of the sidebar */
    background-color: white; /* Set the background color to white */
    z-index: 100; /* Higher z-index to stay above other content */
    padding-top: 20px; /* Adjust padding as needed */
    padding-bottom: 10px;
    border-bottom: 1px solid #d9d9d9;
    /*box-shadow: 0px 2px 5px rgba(0, 0, 0, 0.2); /* Optional: Adds shadow for better separation */
  }
  .progress-bar-container {
    display: flex;
    border: 1px solid black; /* Black outline for the entire bar */
    height: 17px; /* Adjust the height as needed */
    width: 65%; /* Set the width to fit the content */
    margin-left: auto; /* Auto margin on the left */
    margin-right: auto;
  }
  .progress-bar-selected {
    background-color: gold; /* Blue for selected portion */
    opacity: 1; /* Fully opaque */
  }
  .progress-bar-nonselected {
    background-color: gold; /* Same color but lighter for non-selected portion */
    opacity: 0.3; /* Semi-transparent */
  }
  .view-mode-selector {
    padding-top: 10px;
  }
  .sidebar {
    padding-left: 20px; /* Adjust this to match the left alignment */
    padding-bottom: 90px;
    padding-right: 20px;
    position: fixed;
    right: 0;
    top: 0;
    width: 20%;
    height: 100vh; /* Full height of the viewport */
    overflow-y: auto; /* Scrollbar if content overflows */
    background-color: white; /* Set the background to white */
    z-index: 10; /* Ensure sidebar is above other content */
    border-left: 1px solid #d9d9d9;
  }
  .numeric {
    min-width: 92px; /* Adjust as needed to prevent truncation */
  }
  .heading-container {
    display: flex;
    align-items: center;
    justify-content: center;
    width: 70px; /* Take up the full width of the table header cell */
  }
  .heading {
    /*max-width: calc(100% - 9px); /* Adjust based on the size of your SVG */
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
    line-height: 1.2;
    padding: 2px 0;
  }
  .sort-icon {
    margin-left: 6px;
    width: 10px; /* Uniform size for all SVG icons */
    height: 10px; /* Uniform size for all SVG icons */
    flex-shrink: 0; /* Prevent the icon from shrinking */
    fill: currentColor;
    opacity: 0.8; /* Adjust opacity for inactive sort icons */
  }
  .sticker-dark {
    color: white;
  }
  /*.sticker:hover {
    border: 1px solid black;
  }*/
  .sticker {
    display: inline-block;
    width: 60px; /* Fixed width */
    margin-right: 4px;
    margin-left: 4px;
    margin-bottom: 2px;
    padding: 2px 8px 4px 8px;
    border-radius: 15px;
    text-align: center; /* Center the text */
    overflow: hidden; /* Ensures text doesn't overflow */
    text-overflow: ellipsis; /* Adds an ellipsis for overflowed text */
    white-space: nowrap; /* Prevents text wrapping */
    cursor: pointer;
  }
  .freq {
    margin-left: 5px;
  }
  .bar {
    position: absolute;
    height: 20px;
    cursor: default;
  }
  .deviation-container {
    cursor: default;
    display: inline-block;
    position: relative;
    height: 20px;
    width: 80px;
    /*width: 100%; /* Set to the maximum width you want for the deviation bars */
  }
  .bar.negative {
    right: 40px; /* Start from the middle and extend to the left */
    /*transform: translateX(0%);*/
    background-color: #ea4335;
    margin-top: -3.5px;
  }
  .bar.positive {
    left: 40px; /* Start from the middle and extend to the right */
    /*transform: translateX(-10%);*/
    background-color: #4285f4;
    margin-top: -3.5px;
  }
  table {
    /*width: 100%; /* Ensure the table spans the full width */
    border-collapse: collapse; /* Collapse borders for a cleaner look */
    /*margin: 10px auto; /* Center the table */
    table-layout: fixed;
  }
  th {
    cursor: pointer;
    text-align: center; /* Center-align text in headers and cells */
    padding: 10px; /* Consistent padding for headers and cells */
    /*width: 80px; /* Fixed width for each header */
    font-weight: 600;
    white-space: nowrap; /* Prevent text wrapping */
    overflow: hidden; /* Hide overflow */
    text-overflow: ellipsis; /* Add ellipsis to overflowed text */
  }
  td {
    user-select: none;
    padding: 2.5px;
  }
  button {
    display: block; /* Block level element */
    margin-left: auto; /* Auto margins for horizontal centering */
    margin-right: auto;
    margin-top: 10px;
    margin-bottom: 10px;
    cursor: pointer; /* Cursor change to indicate clickability */
  }
  .special-category-label {
    color: red;
  }
  .bold-category-label {
    font-weight: bold;
  }
  .frequency-bar-container {
    cursor: default;
    position: relative;
    top: -8.5px;
  }
  .frequency-label {
    cursor: default;
  }
  /* This has no effect whatsoever... */
  .frequency-heading-container {
    margin-left: 20px;
    text-align: center; /* Center the text inside the container */
  }

  .buttons {
    user-select: none;
    position: sticky;
    bottom: 0; /* Stick to the bottom of the sidebar */
    background-color: white; /* Match background color of sidebar */
    padding: 10px; /* Adjust padding as needed */
    border-top: 1px solid #d9d9d9;
  }

  button.disabled {
    cursor: default;
  }
</style>
