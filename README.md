<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Student Progress Tracker</title>
    <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
    <script src="https://www.gstatic.com/firebasejs/9.6.1/firebase-app.js"></script>
    <script src="https://www.gstatic.com/firebasejs/9.6.1/firebase-firestore.js"></script>
    <style>
        body { font-family: Arial, sans-serif; margin: 20px; }
        h1 { text-align: center; }
        .student-section { margin-top: 20px; padding: 10px; border: 1px solid #ddd; background-color: #f9f9f9; position: relative; }
        canvas { max-width: 100%; height: 300px; }
        .test-item { margin-top: 5px; }
        .hidden { display: none; }
        .test-series-container { padding: 10px; border: 1px solid #ddd; margin-top: 10px; cursor: pointer; }
        .test-series-header { font-weight: bold; cursor: pointer; }
        .divider { height: 1px; background-color: #ddd; margin: 10px 0; }
    </style>
</head>
<body>

    <h1>Gurukul Prasikshan Sansthan - Progress Tracker</h1>

    <div>
        <label for="student-name">Student Name:</label>
        <input type="text" id="student-name" placeholder="Enter student name">
        <button onclick="addStudent()">Add Student</button>
    </div>

    <div id="search-container">
        <label for="search-student">Search Student:</label>
        <input type="text" id="search-student" placeholder="Enter student name to search">
        <button onclick="searchStudent()">Search</button>
    </div>

    <div id="student-container"></div>

    <script>
        // Firebase Configuration
        const firebaseConfig = {
            apiKey: "AIzaSyBU59r0SIZLzAwn93gRxmFbhrdfxaOAqoI",
            authDomain: "gurukul-prasikshan-sansthan.firebaseapp.com",
            projectId: "gurukul-prasikshan-sansthan",
            storageBucket: "gurukul-prasikshan-sansthan.firebasestorage.app",
            messagingSenderId: "493017428949",
            appId: "1:493017428949:web:84854c3b85cfe667b9035c",
            measurementId: "G-CWQL11DMXM"
        };

        // Initialize Firebase
        const app = firebase.initializeApp(firebaseConfig);
        const db = firebase.firestore();

        // Add student data to Firestore
        function addStudent() {
            const studentName = document.getElementById("student-name").value.trim();
            if (!studentName) return alert("Please enter a student name.");

            const studentRef = db.collection('students').doc(studentName);
            studentRef.get().then(doc => {
                if (doc.exists) {
                    return alert("Student already exists!");
                } else {
                    studentRef.set({
                        testSeries: []
                    }).then(() => {
                        renderStudentSection(studentName);
                        document.getElementById("student-name").value = "";
                    });
                }
            });
        }

        // Render student section
        function renderStudentSection(name) {
            const container = document.getElementById("student-container");
            const section = document.createElement("div");
            section.className = "student-section";
            section.setAttribute("id", `section-${name}`);
            section.innerHTML = `
                <div class="student-header" onclick="toggleVisibility('${name}')">${name}'s Progress</div>
                <button onclick="deleteStudent('${name}')">Delete Student</button>
                <button onclick="addTestSeries('${name}')">Add New Test Series</button>
                <div id="${name}-test-series-list" class="hidden"></div>
            `;
            container.appendChild(section);

            // Fetch and render test series from Firestore
            db.collection('students').doc(name).get().then(doc => {
                const studentData = doc.data();
                studentData.testSeries.forEach((series, index) => {
                    createTestSeriesBox(name, index, series);
                });
            });
        }

        // Add test series
        function addTestSeries(name) {
            const seriesName = prompt("Enter the name of this test series:");
            if (!seriesName) return alert("Please enter a valid test series name.");

            const testNames = [];
            const marks = [];
            const maxMarks = [];

            let addMoreTests = true;
            while (addMoreTests) {
                const testName = prompt("Enter test name for the series:");
                const mark = parseInt(prompt("Enter marks obtained:"), 10);
                const maxMark = parseInt(prompt("Enter maximum marks:"), 10);

                if (isNaN(mark) || isNaN(maxMark) || mark > maxMark) {
                    alert("Invalid marks! Please try again.");
                    continue;
                }

                testNames.push(testName);
                marks.push(mark);
                maxMarks.push(maxMark);

                addMoreTests = confirm("Do you want to add another test for this series?");
            }

            // Add the test series to Firestore
            db.collection('students').doc(name).update({
                testSeries: firebase.firestore.FieldValue.arrayUnion({
                    testNames, marks, maxMarks, seriesName
                })
            }).then(() => {
                createTestSeriesBox(name, testNames.length - 1, { testNames, marks, maxMarks, seriesName });
            });
        }

        // Create test series box
        function createTestSeriesBox(name, seriesIndex, series) {
            const testSeriesListContainer = document.getElementById(`${name}-test-series-list`);
            const testSeriesBox = document.createElement("div");
            testSeriesBox.className = "test-series-container";

            const seriesBox = document.createElement("div");
            seriesBox.className = "test-series-header";
            seriesBox.textContent = series.seriesName;
            seriesBox.setAttribute("data-index", seriesIndex);
            seriesBox.onclick = function () { toggleTestSeriesVisibility(name, seriesIndex); };
            testSeriesBox.appendChild(seriesBox);

            const addTestButton = document.createElement("button");
            addTestButton.textContent = "Add Test to Series";
            addTestButton.onclick = () => addTestToSeries(name, seriesIndex);
            testSeriesBox.appendChild(addTestButton);

            // Add the test items here
            series.testNames.forEach((test, index) => {
                const testItem = document.createElement("div");
                testItem.className = "test-item";
                testItem.innerHTML = `${test} - Marks: ${series.marks[index]}/${series.maxMarks[index]} <button onclick="deleteTest('${name}', ${seriesIndex}, ${index})">Delete</button>`;
                testSeriesBox.appendChild(testItem);
            });

            testSeriesListContainer.appendChild(testSeriesBox);
        }

        // Add test to series
        function addTestToSeries(name, seriesIndex) {
            const testNames = studentData[name].testSeries[seriesIndex].testNames;
            const marks = studentData[name].testSeries[seriesIndex].marks;
            const maxMarks = studentData[name].testSeries[seriesIndex].maxMarks;

            const testName = prompt("Enter test name to add:");
            const mark = parseInt(prompt("Enter marks obtained:"), 10);
            const maxMark = parseInt(prompt("Enter maximum marks:"), 10);

            if (isNaN(mark) || isNaN(maxMark) || mark > maxMark) {
                alert("Invalid marks! Please try again.");
                return;
            }

            testNames.push(testName);
            marks.push(mark);
            maxMarks.push(maxMark);

            saveData();
            renderCharts(name, seriesIndex);
        }

        function renderCharts(name, seriesIndex) {
            const testSeries = studentData[name].testSeries[seriesIndex];
            const { testNames, marks, seriesName } = testSeries;

            const chartContainer = document.createElement("div");
            chartContainer.className = "test-series-chart-container";
            const canvas = document.createElement("canvas");
            canvas.setAttribute("id", `chart-${name}-${seriesIndex}`);
            chartContainer.appendChild(canvas);

            const chartData = {
                labels: testNames,
                datasets: [{
                    label: seriesName,
                    data: marks,
                    backgroundColor: testNames.map(() => `rgba(${Math.floor(Math.random() * 255)}, ${Math.floor(Math.random() * 255)}, ${Math.floor(Math.random() * 255)}, 0.6)`),
                    borderColor: testNames.map(() => `rgba(0, 0, 0, 1)`),
                    borderWidth: 1
                }]
            };

            const chartOptions = {
                responsive: true,
                scales: {
                    y: {
                        beginAtZero: true,
                        title: { display: true, text: "Marks" }
                    },
                    x: {
                        title: { display: true, text: "Tests" }
                    }
                }
            };

            // Create the chart
            const ctx = canvas.getContext("2d");
            new Chart(ctx, {
                type: 'bar',
                data: chartData,
                options: chartOptions
            });

            const testSeriesBox = document.querySelector(`#${name}-test-series-list .test-series-container:nth-child(${seriesIndex + 1})`);
            testSeriesBox.appendChild(chartContainer);
        }

        function toggleVisibility(name) {
            const section = document.getElementById(`section-${name}`);
            const testSeriesList = document.getElementById(`${name}-test-series-list`);
            testSeriesList.classList.toggle("hidden");
        }

        function toggleTestSeriesVisibility(name, seriesIndex) {
            const testSeriesListContainer = document.getElementById(`${name}-test-series-list`);
            const testSeriesBoxes = testSeriesListContainer.querySelectorAll(".test-series-container");
            testSeriesBoxes.forEach((box, index) => {
                if (index !== seriesIndex) {
                    const chartContainer = box.querySelector(".test-series-chart-container");
                    if (chartContainer) {
                        chartContainer.classList.add("hidden");
                    }
                }
            });

            const chartContainer = testSeriesBoxes[seriesIndex].querySelector(".test-series-chart-container");
            if (chartContainer) {
                chartContainer.classList.toggle("hidden");
            } else {
                renderCharts(name, seriesIndex);
            }
        }

        function deleteTest(name, seriesIndex, testIndex) {
            if (confirm("Are you sure you want to delete this test?")) {
                const studentRef = db.collection('students').doc(name);
                const studentData = studentRef.get();
                studentData.then(doc => {
                    let testSeries = doc.data().testSeries;
                    testSeries[seriesIndex].testNames.splice(testIndex, 1);
                    testSeries[seriesIndex].marks.splice(testIndex, 1);
                    testSeries[seriesIndex].maxMarks.splice(testIndex, 1);
                    studentRef.update({ testSeries });
                });
            }
        }

        function searchStudent() {
            const searchTerm = document.getElementById("search-student").value.trim();
            if (!searchTerm) return alert("Please enter a name to search.");
            const section = document.getElementById(`section-${searchTerm}`);
            if (section) {
                section.scrollIntoView({ behavior: "smooth" });
            } else {
                alert("Student not found.");
            }
        }

        function deleteStudent(name) {
            if (confirm(`Are you sure you want to delete ${name}'s records?`)) {
                const studentRef = db.collection('students').doc(name);
                studentRef.delete().then(() => {
                    const section = document.getElementById(`section-${name}`);
                    section.remove();
                });
            }
        }
    </script>

</body>
</html><!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Student Progress Tracker</title>
    <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
    <script src="https://www.gstatic.com/firebasejs/9.6.1/firebase-app.js"></script>
    <script src="https://www.gstatic.com/firebasejs/9.6.1/firebase-firestore.js"></script>
    <style>
        body { font-family: Arial, sans-serif; margin: 20px; }
        h1 { text-align: center; }
        .student-section { margin-top: 20px; padding: 10px; border: 1px solid #ddd; background-color: #f9f9f9; position: relative; }
        canvas { max-width: 100%; height: 300px; }
        .test-item { margin-top: 5px; }
        .hidden { display: none; }
        .test-series-container { padding: 10px; border: 1px solid #ddd; margin-top: 10px; cursor: pointer; }
        .test-series-header { font-weight: bold; cursor: pointer; }
        .divider { height: 1px; background-color: #ddd; margin: 10px 0; }
    </style>
</head>
<body>

    <h1>Gurukul Prasikshan Sansthan - Progress Tracker</h1>

    <div>
        <label for="student-name">Student Name:</label>
        <input type="text" id="student-name" placeholder="Enter student name">
        <button onclick="addStudent()">Add Student</button>
    </div>

    <div id="search-container">
        <label for="search-student">Search Student:</label>
        <input type="text" id="search-student" placeholder="Enter student name to search">
        <button onclick="searchStudent()">Search</button>
    </div>

    <div id="student-container"></div>

    <script>
        // Firebase Configuration
        const firebaseConfig = {
            apiKey: "AIzaSyBU59r0SIZLzAwn93gRxmFbhrdfxaOAqoI",
            authDomain: "gurukul-prasikshan-sansthan.firebaseapp.com",
            projectId: "gurukul-prasikshan-sansthan",
            storageBucket: "gurukul-prasikshan-sansthan.firebasestorage.app",
            messagingSenderId: "493017428949",
            appId: "1:493017428949:web:84854c3b85cfe667b9035c",
            measurementId: "G-CWQL11DMXM"
        };

        // Initialize Firebase
        const app = firebase.initializeApp(firebaseConfig);
        const db = firebase.firestore();

        // Add student data to Firestore
        function addStudent() {
            const studentName = document.getElementById("student-name").value.trim();
            if (!studentName) return alert("Please enter a student name.");

            const studentRef = db.collection('students').doc(studentName);
            studentRef.get().then(doc => {
                if (doc.exists) {
                    return alert("Student already exists!");
                } else {
                    studentRef.set({
                        testSeries: []
                    }).then(() => {
                        renderStudentSection(studentName);
                        document.getElementById("student-name").value = "";
                    });
                }
            });
        }

        // Render student section
        function renderStudentSection(name) {
            const container = document.getElementById("student-container");
            const section = document.createElement("div");
            section.className = "student-section";
            section.setAttribute("id", `section-${name}`);
            section.innerHTML = `
                <div class="student-header" onclick="toggleVisibility('${name}')">${name}'s Progress</div>
                <button onclick="deleteStudent('${name}')">Delete Student</button>
                <button onclick="addTestSeries('${name}')">Add New Test Series</button>
                <div id="${name}-test-series-list" class="hidden"></div>
            `;
            container.appendChild(section);

            // Fetch and render test series from Firestore
            db.collection('students').doc(name).get().then(doc => {
                const studentData = doc.data();
                studentData.testSeries.forEach((series, index) => {
                    createTestSeriesBox(name, index, series);
                });
            });
        }

        // Add test series
        function addTestSeries(name) {
            const seriesName = prompt("Enter the name of this test series:");
            if (!seriesName) return alert("Please enter a valid test series name.");

            const testNames = [];
            const marks = [];
            const maxMarks = [];

            let addMoreTests = true;
            while (addMoreTests) {
                const testName = prompt("Enter test name for the series:");
                const mark = parseInt(prompt("Enter marks obtained:"), 10);
                const maxMark = parseInt(prompt("Enter maximum marks:"), 10);

                if (isNaN(mark) || isNaN(maxMark) || mark > maxMark) {
                    alert("Invalid marks! Please try again.");
                    continue;
                }

                testNames.push(testName);
                marks.push(mark);
                maxMarks.push(maxMark);

                addMoreTests = confirm("Do you want to add another test for this series?");
            }

            // Add the test series to Firestore
            db.collection('students').doc(name).update({
                testSeries: firebase.firestore.FieldValue.arrayUnion({
                    testNames, marks, maxMarks, seriesName
                })
            }).then(() => {
                createTestSeriesBox(name, testNames.length - 1, { testNames, marks, maxMarks, seriesName });
            });
        }

        // Create test series box
        function createTestSeriesBox(name, seriesIndex, series) {
            const testSeriesListContainer = document.getElementById(`${name}-test-series-list`);
            const testSeriesBox = document.createElement("div");
            testSeriesBox.className = "test-series-container";

            const seriesBox = document.createElement("div");
            seriesBox.className = "test-series-header";
            seriesBox.textContent = series.seriesName;
            seriesBox.setAttribute("data-index", seriesIndex);
            seriesBox.onclick = function () { toggleTestSeriesVisibility(name, seriesIndex); };
            testSeriesBox.appendChild(seriesBox);

            const addTestButton = document.createElement("button");
            addTestButton.textContent = "Add Test to Series";
            addTestButton.onclick = () => addTestToSeries(name, seriesIndex);
            testSeriesBox.appendChild(addTestButton);

            // Add the test items here
            series.testNames.forEach((test, index) => {
                const testItem = document.createElement("div");
                testItem.className = "test-item";
                testItem.innerHTML = `${test} - Marks: ${series.marks[index]}/${series.maxMarks[index]} <button onclick="deleteTest('${name}', ${seriesIndex}, ${index})">Delete</button>`;
                testSeriesBox.appendChild(testItem);
            });

            testSeriesListContainer.appendChild(testSeriesBox);
        }

        // Add test to series
        function addTestToSeries(name, seriesIndex) {
            const testNames = studentData[name].testSeries[seriesIndex].testNames;
            const marks = studentData[name].testSeries[seriesIndex].marks;
            const maxMarks = studentData[name].testSeries[seriesIndex].maxMarks;

            const testName = prompt("Enter test name to add:");
            const mark = parseInt(prompt("Enter marks obtained:"), 10);
            const maxMark = parseInt(prompt("Enter maximum marks:"), 10);

            if (isNaN(mark) || isNaN(maxMark) || mark > maxMark) {
                alert("Invalid marks! Please try again.");
                return;
            }

            testNames.push(testName);
            marks.push(mark);
            maxMarks.push(maxMark);

            saveData();
            renderCharts(name, seriesIndex);
        }

        function renderCharts(name, seriesIndex) {
            const testSeries = studentData[name].testSeries[seriesIndex];
            const { testNames, marks, seriesName } = testSeries;

            const chartContainer = document.createElement("div");
            chartContainer.className = "test-series-chart-container";
            const canvas = document.createElement("canvas");
            canvas.setAttribute("id", `chart-${name}-${seriesIndex}`);
            chartContainer.appendChild(canvas);

            const chartData = {
                labels: testNames,
                datasets: [{
                    label: seriesName,
                    data: marks,
                    backgroundColor: testNames.map(() => `rgba(${Math.floor(Math.random() * 255)}, ${Math.floor(Math.random() * 255)}, ${Math.floor(Math.random() * 255)}, 0.6)`),
                    borderColor: testNames.map(() => `rgba(0, 0, 0, 1)`),
                    borderWidth: 1
                }]
            };

            const chartOptions = {
                responsive: true,
                scales: {
                    y: {
                        beginAtZero: true,
                        title: { display: true, text: "Marks" }
                    },
                    x: {
                        title: { display: true, text: "Tests" }
                    }
                }
            };

            // Create the chart
            const ctx = canvas.getContext("2d");
            new Chart(ctx, {
                type: 'bar',
                data: chartData,
                options: chartOptions
            });

            const testSeriesBox = document.querySelector(`#${name}-test-series-list .test-series-container:nth-child(${seriesIndex + 1})`);
            testSeriesBox.appendChild(chartContainer);
        }

        function toggleVisibility(name) {
            const section = document.getElementById(`section-${name}`);
            const testSeriesList = document.getElementById(`${name}-test-series-list`);
            testSeriesList.classList.toggle("hidden");
        }

        function toggleTestSeriesVisibility(name, seriesIndex) {
            const testSeriesListContainer = document.getElementById(`${name}-test-series-list`);
            const testSeriesBoxes = testSeriesListContainer.querySelectorAll(".test-series-container");
            testSeriesBoxes.forEach((box, index) => {
                if (index !== seriesIndex) {
                    const chartContainer = box.querySelector(".test-series-chart-container");
                    if (chartContainer) {
                        chartContainer.classList.add("hidden");
                    }
                }
            });

            const chartContainer = testSeriesBoxes[seriesIndex].querySelector(".test-series-chart-container");
            if (chartContainer) {
                chartContainer.classList.toggle("hidden");
            } else {
                renderCharts(name, seriesIndex);
            }
        }

        function deleteTest(name, seriesIndex, testIndex) {
            if (confirm("Are you sure you want to delete this test?")) {
                const studentRef = db.collection('students').doc(name);
                const studentData = studentRef.get();
                studentData.then(doc => {
                    let testSeries = doc.data().testSeries;
                    testSeries[seriesIndex].testNames.splice(testIndex, 1);
                    testSeries[seriesIndex].marks.splice(testIndex, 1);
                    testSeries[seriesIndex].maxMarks.splice(testIndex, 1);
                    studentRef.update({ testSeries });
                });
            }
        }

        function searchStudent() {
            const searchTerm = document.getElementById("search-student").value.trim();
            if (!searchTerm) return alert("Please enter a name to search.");
            const section = document.getElementById(`section-${searchTerm}`);
            if (section) {
                section.scrollIntoView({ behavior: "smooth" });
            } else {
                alert("Student not found.");
            }
        }

        function deleteStudent(name) {
            if (confirm(`Are you sure you want to delete ${name}'s records?`)) {
                const studentRef = db.collection('students').doc(name);
                studentRef.delete().then(() => {
                    const section = document.getElementById(`section-${name}`);
                    section.remove();
                });
            }
        }
    </script>

</body>
</html>
