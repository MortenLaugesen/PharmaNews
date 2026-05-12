IF YEAR([Online Initial Date]) <= YEAR(TODAY())
THEN [Volume Total]
END


IF YEAR([Online Initial Date]) > YEAR(TODAY())
THEN [Volume Total]
END
