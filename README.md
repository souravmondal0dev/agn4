-- Start a transaction
BEGIN TRANSACTION;

-- Step 1: Clear previous results (optional for reruns)
DELETE FROM Allotments;
DELETE FROM UnallotedStudents;

-- Step 2: Create a working temp table for remaining seats
-- This avoids updating the original SubjectDetails directly
IF OBJECT_ID('tempdb..#TempSubjectSeats') IS NOT NULL
    DROP TABLE #TempSubjectSeats;

SELECT SubjectId, RemainingSeats
INTO #TempSubjectSeats
FROM SubjectDetails;

-- Step 3: Cursor to loop over students in descending GPA order
DECLARE @StudentId VARCHAR(50);
DECLARE @SubjectId VARCHAR(50);
DECLARE @Preference INT;
DECLARE @SeatsLeft INT;
DECLARE @Allotted BIT;

DECLARE student_cursor CURSOR FOR
SELECT StudentId
FROM StudentDetails
ORDER BY GPA DESC;

OPEN student_cursor;
FETCH NEXT FROM student_cursor INTO @StudentId;

WHILE @@FETCH_STATUS = 0
BEGIN
    SET @Allotted = 0;

    -- Check preferences in order 1 to 5
    DECLARE pref_cursor CURSOR FOR
    SELECT SubjectId, Preference
    FROM StudentPreference
    WHERE StudentId = @StudentId
    ORDER BY Preference ASC;

    OPEN pref_cursor;
    FETCH NEXT FROM pref_cursor INTO @SubjectId, @Preference;

    WHILE @@FETCH_STATUS = 0 AND @Allotted = 0
    BEGIN
        -- Check available seats
        SELECT @SeatsLeft = RemainingSeats
        FROM #TempSubjectSeats
        WHERE SubjectId = @SubjectId;

        IF @SeatsLeft > 0
        BEGIN
            -- Allot this subject
            INSERT INTO Allotments (SubjectId, StudentId)
            VALUES (@SubjectId, @StudentId);

            -- Update seat count
            UPDATE #TempSubjectSeats
            SET RemainingSeats = RemainingSeats - 1
            WHERE SubjectId = @SubjectId;

            SET @Allotted = 1;
        END

        FETCH NEXT FROM pref_cursor INTO @SubjectId, @Preference;
    END

    CLOSE pref_cursor;
    DEALLOCATE pref_cursor;

    -- If not allotted any subject
    IF @Allotted = 0
    BEGIN
        INSERT INTO UnallotedStudents (StudentId)
        VALUES (@StudentId);
    END

    FETCH NEXT FROM student_cursor INTO @StudentId;
END

CLOSE student_cursor;
DEALLOCATE student_cursor;

-- Commit the transaction
COMMIT;
