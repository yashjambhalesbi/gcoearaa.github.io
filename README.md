
<!DOCTYPE html>
<html>
<head>
	<title>Student Attendance Tracker</title>
</head>
<body>
	<h1>Student Attendance Tracker</h1>
	<form method="post" action="submit-attendance.php">
		<label for="date">Date:</label>
		<input type="date" name="date" required>
		<br>
		<label for="student">Student:</label>
		<select name="student" required>
			<?php
			// Connect to database
			$conn = mysqli_connect("localhost", "username", "password", "database");

			// Retrieve list of students from database
			$query = "SELECT * FROM students ORDER BY name";
			$result = mysqli_query($conn, $query);
			while ($row = mysqli_fetch_assoc($result)) {
				echo "<option value=\"" . $row["id"] . "\">" . $row["name"] . "</option>";
			}

			// Close database connection
			mysqli_close($conn);
			?>
		</select>
		<br>
		<label for="status">Attendance:</label>
		<select name="status" required>
			<option value="present">Present</option>
			<option value="absent">Absent</option>
			<option value="tardy">Tardy</option>
		</select>
		<br>
		<input type="submit" value="Submit Attendance">
	</form>
	<br>
	<a href="generate-report.php">Generate Monthly Report</a>
</body>
</html>
<?php
// Retrieve data from form
$date = $_POST["date"];
$student = $_POST["student"];
$status = $_POST["status"];

// Connect to database
$conn = mysqli_connect("localhost", "username", "password", "database");

// Insert attendance record into database
$query = "INSERT INTO attendance (date, student_id, status) VALUES ('$date', '$student', '$status')";
mysqli_query($conn, $query);

// Close database connection
mysqli_close($conn);

// Redirect to previous page
header("Location: {$_SERVER['HTTP_REFERER']}");
exit();
?>
<?php
// Connect to database
$conn = mysqli_connect("localhost", "username", "password", "database");

// Retrieve list of students from database
$query = "SELECT * FROM students ORDER BY name";
$result = mysqli_query($conn, $query);

// Loop through list of students and generate report for each one
while ($row = mysqli_fetch_assoc($result)) {
	// Retrieve attendance records for current student and current month
	$student_id = $row["id"];
	$current_month = date("Y-m");
	$query2 = "SELECT * FROM attendance WHERE student_id = '$student_id' AND date LIKE '$current_month%'";
	$result2 = mysqli_query($conn, $query2);

	// Calculate attendance percentage for current student and current month
	$total_days = date("t");
	$present_days = 0;
	while ($row2 = mysqli_fetch_assoc($result2)) {
		if ($row2["status"] == "present") {
			$present_days++;
		}
	}
	$attendance_percentage = ($present_days / $total_days) * 100;

	// Retrieve parent's email and phone number for current student
	$parent_email = $row["parent_email"];
	$parent_phone = $row["parent_phone"];

	// Compose email message with monthly attendance report
	$subject = "Monthly Attendance Report for " . $row["name"];
	$message = "Dear parent,\n\nThis is your child's monthly attendance report for " . date("F Y") . ":\n\n";
	$message .= "Total days: " . $total_days . "\n";
	$message .= "Present days: " . $present_days . "\n";
	$message .= "Attendance percentage: " . number_format($attendance_percentage, 2) . "%\n\n";
	$message .= "Thank you,\nSchool Administration";

	// Send email to parent
	$headers = "From: School Administration <admin@school.com>\r\n";
	mail($parent_email, $subject, $message, $headers);

	// Send SMS message to parent
	$sms_message = "Monthly attendance report for " . $row["name"] . ": " . number_format($attendance_percentage, 2) . "% attendance";
	$ch = curl_init();
	curl_setopt($ch, CURLOPT_URL, "https://api.smsprovider.com/send?to=" . $parent_phone . "&message=" . urlencode($sms_message));
	curl_setopt($ch, CURLOPT_RETURNTRANSFER, 1);
	$output = curl_exec($ch);
	curl_close($ch);
}

// Close database connection
mysqli_close($conn);

// Redirect to previous page
header("Location: {$_SERVER['HTTP_REFERER']}");
exit();
?>
