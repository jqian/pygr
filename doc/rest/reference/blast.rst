:mod:`blast` --- BLAST mapping interfaces
=============================================

.. module:: blast
   :synopsis: BLAST mapping interfaces.
.. moduleauthor:: Christopher Lee <leec@chem.ucla.edu>
.. sectionauthor:: Christopher Lee <leec@chem.ucla.edu>


This module provides two main classes for performing BLAST or MEGABLAST
searches on a sequence database.

External Requirements
---------------------
This module makes use of several external programs:

* *NCBI toolkit*: The BLAST database functionality in this module
  requires that the NCBI toolkit
  be installed on your system.  Specifically, some functions will call the command line
  programs ``formatdb``, ``blastall``, and ``megablast``.
  
* *RepeatMasker*: :class:`MegablastMapping` calls the command line
  program ``RepeatMasker`` to mask out repetitive sequences from seeding alignments,
  but to allow extension of alignments into masked regions.
  


BlastMapping
------------
This class provides a mapping interface for searching a sequence
database with BLAST.  You simply create the mapping object with the
target database you want it to search, then use it as a mapping.

Here's a simple example of opening a BLAST database and searching it for matches to a specific piece of sequence::

   db = SequenceFileDB('sp') # OPEN SWISSPROT SEQUENCE DB
   blastmap = BlastMapping(db) # CREATE A MAPPING OBJECT FOR RUNNING BLAST
   m = blastmap[db['CYGB_HUMAN'][40:-40]] # DO BLAST SEARCH
   try:
       hit = m[db['MYG_CHICK']] # IS MYG_CHICK HOMOLOGOUS?
       print 'Identity:', hit.pIdentity()*100, 'Aligned:', hit.pAligned()*100
       for src,dest in hit.items(): # PRINT MATCHING INTERVALS
           print src, repr(src), '\n', dest, repr(dest), '\n'
   except KeyError:
       print 'MYG_CHICK not homologous.'


Let's go through this example line by line:


  
* We can create a BlastMapping for any sequence database.  BlastMapping
  will figure out what type of sequences (nucleotide vs. protein) are
  in the database, in order to construct the BLAST index file properly.
  
* db['CYGB_HUMAN'] obtains a sequence object representing the SwissProt sequence whose ID is CYGB_HUMAN.  The slice operation [40:-40] behaves just like normal Python slicing: it obtains a sequence object representing the subinterval omitting the first 40 letters and last 40 letters of the sequence.
  
* Querying the blast mapping object with a sequence object as a key
  searches the BLAST database for homologies to it, using NCBI BLAST.  It chooses reasonable parameters based upon the sequence types of the database and supplied query.
  
* You can specify search parameter options if you wish, by calling
  the blast mapping as a function with the sequence object and
  your extra parameters.  It returns a Pygr sequence mapping (multiple alignment) that represents a standard Pygr graph of alignment relationships between s and the homologies that were found.  Since this mode is designed for being
  able to save alignments for multiple sequences in a single multiple
  sequence alignment, when reading this alignment you need to specify
  which source sequence you want alignment information for, e.g.::
  
     msa = blastmap(db['CYGB_HUMAN'][40:-40], expmax=1e-10)
     m = msa[db['CYGB_HUMAN']]
  
  The rest of the code would be identical to the example above.
  
* The expression m[db['MYG_CHICK']] obtains the "edge information" for the graph relationship between the two sequence nodes s and MYG_CHICK.  (if there was no edge in the m graph representing a relationship between these two sequences, this would produce a
  :exc:`KeyError`).  This edge information consists of a set of interval alignment relationships, which are printed out in this example.
  


.. class:: BlastMapping(seqDB, filepath=None, blastReady=False, blastIndexPath=None, blastIndexDirs=None, **kwargs)

   * *seqDB*: the sequence database to search via BLAST.

   * *filepath*: location of a FASTA file to initialize from (optional;
     if not provided, BlastMapping will try to obtain it from the
     *seqDB* object).

   * *blastReady* option specifies whether BLAST index files should
     be immediately
     constructed (using ``formatdb``).  Note, if you ask it to generate
     BLAST results, it will automatically create the index files for you
     if they are missing.

   * *blastIndexPath*, if not None, specifies the path to the BLAST index
     files for this database.  For example, if the BLAST index files are
     ``/some/path/foo.psd`` etc., then *blastIndexPath*``='/some/path/foo'``.

   * *blastIndexDirs*, if not None, specifies a list of directories in which
     to search for and create BLAST index files.  Entries in the list can be
     either a string, or a function that takes no parameters and returns
     a string path.  A string value "FILEPATH" instructs it to use the
     filepath of the FASTA file associated with the BlastDB.
     The default value of this attribute on the :class:`BlastDB` class is::

        ['FILEPATH',os.getcwd,os.path.expanduser,pygr.classutil.default_tmp_path]

     This corresponds to: self.filepath, current directory, the user's HOME
     directory, and the default temporary directory used by the Python
     function :meth:`os.tempnam()`.


Useful methods:
.. method:: formatdb(filepath=None)

   Triggers the BlastMapping to construct new BLAST index files, either at the
   location specified by *filepath*, if not None, or in the first
   directory in the :attr:`blastIndexDirs` list where the index files
   can be succesfully built.  Index files are generated using the
   "formatdb" program provided by NCBI, which must be in your
   PATH for this method to work.


.. method:: __call__(seq, al=None, blastpath='blastall', blastprog=None, expmax=0.001, maxseq=None, opts=", verbose=True)

   run a BLAST search on sequence object seq.
   *maxseq* will limit the number of returned hits to the best *maxseq* hits.
   *al* if not None, must be an alignment object in which you want the results
   to be saved.  Note: in this case, the :meth:`blast` function will not automatically
   call the alignment's :meth:`build()` method; you will have to do that yourself.
   *blastpath* gives the command to run BLAST.
   *blastprog*, if not None, should be a string giving the name of the BLAST
   program variant you wish to run, e.g. 'blastp' or 'blastn' etc.  If None,
   this will be figured out automatically based on the sequence type of *seq*
   and of the sequences in this database.
   *expmax* should be a float value giving the largest "expectation score"
   you wish to allow homology to be reported for.
   *opts* allows you to specify arbitrary command line arguments to the BLAST
   program, for controlling its search parameters.
   *verbose=False* allows you to switch off printing of explanatory messages to
   stderr.




Useful attributes:

  
.. attribute:: filepath

   the location of the FASTA sequence file upon which
   this :class:`BlastMapping` is based.
  
.. attribute:: blastIndexPath

   if present, the location of the BLAST index files
   associated with this :class:`BlastMapping`.  If not present, the location is assumed
   to be the same as the FASTA file.
  
.. attribute:: blastIndexDirs

   the list of directories in which to search for
   or build BLAST index files for this :class:`BlastMapping`.  For details, see
   the explanation for the constructor method, above.
  



MegablastMapping
----------------
This class provides a mapping interface for searching a sequence
database with MEGABLAST.  You use it just like a :class:`BlastMapping` object.
It accepts some different arguments when you use a megablast mapping
object as a callable:

.. method:: __call__(seq, al=None, blastpath='megablast', expmax=1e-20, maxseq=None, minIdentity=None, maskOpts='-U T -F m', rmPath='RepeatMasker', rmOpts='-xsmall', opts=", verbose=True)

   first performs repeat masking on the sequence by converting repeats to lowercase,
   then runs megablast with command line options to prevent seeding new alignments
   within repeats, but allowing extension of alignments into repeats.
   In addition to the blast options (described above),
   *minIdentity* should be a number (maximum value, 100)
   indicating the minimum percent identity for hits to be returned.
   *rmPath* gives the command to use to run RepeatMasker.
   *rmOpts* allows you to give command line options to RepeatMasker.
   The default setting causes RepeatMasker to mark repetitive regions in the
   query in lowercase, which then works in concert with the *maskOpts* option, next.
   *maskOpts* gives command line options for controlling the megablast program's
   masking behavior.  The default value prevents megablast from using repetitive
   sequence as a seed for starting a hit, but allows it to propagate a regular
   (non-repetitive hit) through a repetitive region.




