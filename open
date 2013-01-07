#!/usr/bin/perl
use strict;
use warnings;

use constant
{
	PROG_REVEAL => 'rox',
	PROG_EDIT   => 'vim',
};

sub open_smart;
sub edit;
sub stdin_to_editor;
sub reveal;
sub header_edit;
sub wait_or_not;

sub usage
{
	print STDERR <<"!";
Usage: $0 -[efWwRh]
-e: edit
-f: stdin-edit
-W: wait for exit (true by default for editing)
-w: don't wait for exit
-R: reveal
-h: header search
!
	exit 1;
}

my $cmd = \&open_smart;
my(@files, @args);

my %opts = (
	'e'    => 0,
	'f'    => 0,
	'W'    => 0,
	'R'    => 0,
	'h'    => 0,
);

my $wait_set = 0;

usage() unless @ARGV;

for(my $i = 0; $i < @ARGV; ++$i){
	$_ = $ARGV[$i];

	if($_ eq '--'){
		push @files, @ARGV[$i + 1 .. $#ARGV];
		last;
	}

	if(/^-([a-z])$/i){
		my $k = $1;

		if(exists $opts{$k}){
			$opts{$k} = 1;
			$wait_set = 1 if $k eq 'W';
		}elsif($k eq 'w'){
			$opts{W} = 0;
			$wait_set = 1;
		}else{
			usage();
		}

	}elsif($_ eq '--args'){
		push @args, @ARGV[$i + 1 .. $#ARGV];
		last;

	}elsif(/^-/){
		usage();

	}else{
		push @files, $_;

	}
}

if($opts{e} + $opts{f} + $opts{R} + $opts{h} > 1){
	print STDERR "Can't combine -e, -f, -R and -h\n";
	usage();
}

my $should_wait = 1;
if($opts{e}){
	$cmd = \&edit;

}elsif($opts{f}){
	# <STDIN> | $EDITOR -
	$cmd = \&stdin_to_editor;

}elsif($opts{R}){
	# open with rox
	$cmd = \&reveal;
	$should_wait = 0;

}elsif($opts{h}){
	# search /usr/include/$_ for @files
	$cmd = \&header_edit;

}

$opts{W} = 1 if $should_wait and not $wait_set;

exit(&{$cmd}((
		wait  => !!$opts{W},
		args  => [@args],
		files => [@files])));

# end ---

sub open_smart
{
	sub read_maps
	{
		my $rc = "$ENV{HOME}/.openrc";
		open F, '<', $rc or die "open $rc: $!\n";

		my %maps;

		my $suffix = 0;
		while(<F>){
			chomp;
			s/#.*//;

			if(/^\[(.*)\]$/){
				if($1 eq 'full'){
					$suffix = 0;
				}elsif($1 eq 'suffix'){
					$suffix = 1;
				}else{
					die "invalid section \"$1\" in $rc\n";
				}

			}elsif(my($prog, $matches) = /^([^:]+): *(.*)/){
				sub getenv
				{
					my $k = shift;
					return $ENV{$k} if $ENV{$k};
					my %backup = (
						"TERM" => "urxvt",
						"VISUAL" => "vim",
					);
					return $backup{$k} if $backup{$k};
					return "\$$k";
				}

				my @matches = split / *, */, $matches;

				$prog =~ s/\$([A-Z_]+)/getenv($1)/e;

				push @{$maps{$prog}}, [ $_, $suffix ] for @matches;

			}elsif(length){
				die "invalid confiuration line: \"$1\" in $rc\n";
			}
		}

		close F;

		return %maps;
	}

	my %maps = read_maps();
	my $ec = 0;
	my %h = @_;

	my @to_open;

file:
	for my $fnam (@{$h{files}}){
		#print "maps:\n";

		if(-d $fnam){
			push @to_open, [($h{wait}, PROG_REVEAL, @{$h{args}}, $fnam)];
			next file;
		}

		for my $prog (keys %maps){
			#print "  $_:\n";
			for(@{$maps{$prog}}){
				my($reg, $suffix) = ($_->[0], $_->[1]);
				if($suffix){
					$reg = "\\.$reg\$";
				}
				#print "    $reg\n"

				if($fnam =~ /$reg/){
					push @to_open, [($h{wait}, $prog, @{$h{args}}, $fnam)];
					next file;
				}
			}
		}

		die "no program found for $fnam\n";
	}

	wait_or_not(@{$_}) for @to_open;

	return 0;
}

sub wait_or_not
{
	my($wait, @rest) = @_;
	my $pid = fork();

	die "fork(): $!\n" unless defined $pid;

	if($pid == 0){
		if($rest[0] =~ / /){
			my $a = shift @rest;
			unshift @rest, split / +/, $a;
		}

		exec @rest;
		die;
	}else{
		# parent
		if($wait){
			my $reaped = wait();
			my $ret = $?;

			die "wait(): $!\n" if $reaped == -1;
			warn "unexpected dead child $reaped (expected $pid)\n" if $reaped != $pid;

			return $ret;
		}
	}
}

sub edit
{
	my %h = @_;
	my $e = $ENV{VISUAL} || $ENV{EDITOR} || PROG_EDIT;
	return wait_or_not($h{wait}, $e, @{$h{args}}, @{$h{files}});
}

sub stdin_to_editor
{
	my $tmp = "/tmp/stdin_$$";

	open F, '>', $tmp or die "open $tmp: $!\n";
	print F $_ while <STDIN>;
	close F;

	my %h = @_;
	push @{$h{files}}, $tmp;
	my $r = edit(%h);
	unlink $tmp;
	return $r;
}

sub reveal
{
	my %h = @_;
	return wait_or_not($h{wait}, PROG_REVEAL, @{$h{args}}, @{$h{files}});
}

sub header_edit
{
	my %h = @_;
	my @files = @{$h{files}};
	@{$h{files}} = ();

	for my $name (@files){
		sub find_header
		{
			my @inc = ("", "arpa", "net", "sys");
			my $r = shift;
			my @matches;

			for(my @tmp = @inc){
				push @inc, "x86_64-linux-gnu/$_";
			}

			for my $inc (@inc){
				$inc = "/usr/include/$inc";

				opendir D, $inc or next;
				push @matches, map { "$inc/$_" } grep /$r/, readdir D;
				closedir D;
			}

			return @matches;
		}

		my @paths = find_header($name);
		push @{$h{files}}, @paths if @paths;
	}

	return edit(%h);
}
