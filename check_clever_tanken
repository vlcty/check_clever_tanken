#!/usr/bin/perl
use strict;
use LWP::UserAgent;
use Getopt::Long;

my $stationID = 0;
my $alarm = {};

# Defines how may lines the fuel type and the price is apart in line counts.
my $priceLineOffset = 10;

sub exitCritical {
    printf("CRITICAL - %s\n", shift);
    exit(2);
}

sub exitUnknown {
    printf("UNKNOWN - %s\n", shift);
    exit(3);
}

sub substituteSpecialCharacters {
    my $string = shift;

    $string =~ s/&amp;/und/ig;
    $string =~ s/ß/ss/ig;
    $string =~ s/ +/ /ig;
    $string =~ s/ä/ae/ig;
    $string =~ s/ö/ae/ig;
    $string =~ s/ü/ae/ig;
    $string =~ s/@/at/ig;

    return $string;
}

sub parseArguments {
    my @alarmings = ();

    GetOptions(
        'station=i' => \$stationID,
        'alarm=s' => \@alarmings
    );

    if ( $stationID !~ /^\d+$/ ) {
        exitUnknown('Station ID has to be a number');
    }

    if ( $stationID <= 0 ) {
        exitUnknown('Station ID can\'t be below 1');
    }

    foreach my $currentAlarm ( @alarmings ) {
        if ( $currentAlarm =~ m/^(.*): (.*)$/ ) {
            $alarm->{$1} = $2;
        }
    }
}

sub extractFuelType {
    my $line = shift;

    if ( $line =~ /price-type-name">(.*?)<\/div>/ ) {
        return $1;
    }
    else {
        return undef;
    }
}

sub extractFuelPrice {
    my $line = shift;

    if ( $line =~ /current-price-\d">(\d+\.\d+)<\/span>/ ) {
        return $1;
    }
    else {
        exitCritical('Was not able to extract price from line: ' . $line);
    }
}

sub extractStationStreet {
    my $line = shift;

    if ( $line =~ /<span itemprop="streetAddress">(.*?)<\/span>/ ) {
        return substituteSpecialCharacters($1);
    }
    else {
        return undef;
    }
}

sub extractStationName {
    my $line = shift;

    if ( $line =~ /<span class="strong-title" itemprop="name">(.*?)<\/span>/ ) {
        return substituteSpecialCharacters($1);
    }
    else {
        return undef;
    }
}

sub convertToNagiosVariable {
    my $string = shift;
    $string = substituteSpecialCharacters($string);

    $string =~ s/\.//ig;
    $string =~ s/ +/_/ig;
    $string =~ s/-+/_/ig;

    return $string;
}

sub produceUserAgent {
    my $ua = LWP::UserAgent->new();
    $ua->agent('Mozilla/5.0 (Macintosh; Intel Mac OS X 10.14; rv:66.0) Gecko/20100101 Firefox/66.0');
    $ua->default_header('Referer' => 'https://www.clever-tanken.de/');
    $ua->default_header('Accept' => 'text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8');
    $ua->default_header('Accept-Language' => 'de,en-US;q=0.7,en;q=0.3');
    $ua->default_header('Accept-Encoding' => 'gzip, deflate');
    $ua->default_header('Connection' => 'close');
    $ua->default_header('Upgrade-Insecure-Requests' => '1');
    $ua->default_header('DNT' => '1');

    return $ua;
}

sub extractLongestFuelTypeStringLenght {
    my $prices = shift;
    my $observedMaximalLength = 0;

    foreach my $currentFuel ( keys %{$prices} ) {
        my $length = length($currentFuel);

        if ( $length > $observedMaximalLength ) {
            $observedMaximalLength = $length;
        }
    }

    # We now have the length of the longest fuel type string
    # I multiply it with -1 to get a negative number.
    # This is because a negative number does a left alignment of the string and
    # adds the whitespaces after it.
    # Otherwise the whitespaces would be on the left side which looks stupid in
    # my eyes.
    return $observedMaximalLength * -1;
}

sub generateDisplayAndPerformanceString {
    my ( $stationName, $stationStreet, $prices ) = @_;

    my $longestFuelTypeStringLength = extractLongestFuelTypeStringLenght($prices);
    my $displayString = sprintf("%s %s:\n\n", $stationName, $stationStreet);
    my $performanceString = '';
    my $performanceStringStationPrefix = convertToNagiosVariable($stationName . ' ' . $stationStreet);
    my $returnCode = 0;

    foreach my $currentFuel ( sort keys %{$prices} ) {
        # Generate a performance string part
        $performanceString .= sprintf('%s_%s=%.2f ',
            $performanceStringStationPrefix,
            convertToNagiosVariable($currentFuel),
            $prices->{$currentFuel});

        # Generate a display string part depending on a set limit
        if ( exists($alarm->{$currentFuel}) ) {
            # We have a limit set for this fule type. We now have to check if
            # it's exactly the limit or even below the limit.
            my $priceDifference = $prices->{$currentFuel} - $alarm->{$currentFuel};

            if ( $priceDifference == 0 ) {
                $displayString .= sprintf("*%${longestFuelTypeStringLength}s: %.2f €/l -> Limit von %.2f €/l erreicht*\n",
                    $currentFuel,
                    $prices->{$currentFuel},
                    $alarm->{$currentFuel});

                $returnCode = 1;

                next; # We have what we want, so skip the rest
            }
            elsif ( $priceDifference < 0 ) {
                $displayString .= sprintf("*%${longestFuelTypeStringLength}s: %.2f €/l -> %.0f Cent günstiger als das Limit von %.2f €/l*\n",
                    $currentFuel,
                    $prices->{$currentFuel},
                    abs($priceDifference) * 100, # Prices are given in Euros, but we want the difference in cents. 1 € = 100 cents.
                    $alarm->{$currentFuel});

                $returnCode = 1;

                next; # We have what we want, so skip the rest
            }
        }
        else {
            # No limit was defined for this fuel type. So just add it without
            # further inspection
            $displayString .= sprintf("%${longestFuelTypeStringLength}s: %.2f €/l\n",
                $currentFuel, $prices->{$currentFuel});

        }
    }

    printf("%s|%s\n", $displayString, $performanceString);
    exit($returnCode);
}

sub analyzeStationContent {
    my $response = shift;

    my @content = split(/\n/, $response->decoded_content());

    my $stationName = undef;
    my $stationStreet = undef;
    my $prices = {};

    for ( my $currentLine = 0; $currentLine < scalar(@content); $currentLine++ ) {
        my $currentLineString = $content[$currentLine];

        if ( my $name = extractStationName($currentLineString) ) {
            $stationName = substituteSpecialCharacters($name);
            next;
        }

        if ( my $street = extractStationStreet($currentLineString) ) {
            $stationStreet = substituteSpecialCharacters($street);
            next;
        }

        if ( my $fuelType = extractFuelType($currentLineString) ) {
            $prices->{$fuelType} =
                extractFuelPrice($content[$currentLine + $priceLineOffset]);
        }
    }

    if ( length($stationName) == 0 ) {
        exitCritical('Was not able to extract station\'s name');
    }

    if ( length($stationStreet) == 0 ) {
        exitCritical('Was not able to extract station\'s street');
    }

    generateDisplayAndPerformanceString($stationName,
        $stationStreet, $prices);
}

sub main {
    parseArguments();

    my $ua = produceUserAgent();

    my $response = $ua->get('https://www.clever-tanken.de/tankstelle_details/' . $stationID);

    if ( $response->is_success() ) {
        analyzeStationContent($response);
    }
    else {
        exitCritical('Was not able to fetch page for station with id ' . $stationID);
    }
}

main();
